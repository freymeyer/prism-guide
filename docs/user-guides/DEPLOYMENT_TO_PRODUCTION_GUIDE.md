# PRISM-Matutum: Complete Deployment & Production Setup Guide

**Version:** 1.0  
**Last Updated:** January 8, 2026  
**Target Audience:** System Administrators, DevOps Engineers  

---

## 📋 Table of Contents

1. [Introduction](#introduction)
2. [Pre-Installation Checklist](#pre-installation-checklist)
3. [Phase 1: Infrastructure Setup](#phase-1-infrastructure-setup)
4. [Phase 2: Application Installation](#phase-2-application-installation)
5. [Phase 3: Database Configuration](#phase-3-database-configuration)
6. [Phase 4: Initial Data Population](#phase-4-initial-data-population)
7. [Phase 5: User Management Setup](#phase-5-user-management-setup)
8. [Phase 6: KoboToolbox Integration](#phase-6-kobotoolbox-integration)
9. [Phase 7: Production Hardening](#phase-7-production-hardening)
10. [Phase 8: Monitoring & Maintenance](#phase-8-monitoring--maintenance)
11. [Troubleshooting](#troubleshooting)
12. [Quick Reference Commands](#quick-reference-commands)

---

## 1. Introduction

This guide provides a complete step-by-step walkthrough for deploying PRISM-Matutum from scratch to a fully operational production system. By following this guide, you will:

- ✅ Set up the complete infrastructure (PostgreSQL + PostGIS, Redis, Celery)
- ✅ Configure the Django application with proper security settings
- ✅ Populate the database with essential data (taxonomy, species, geographic locations)
- ✅ Create and configure user accounts with proper roles
- ✅ Integrate with KoboToolbox for field data collection
- ✅ Harden the system for production use
- ✅ Set up monitoring and backup systems

### System Overview

**PRISM-Matutum** (Protected Area Biodiversity Monitoring System) is a comprehensive platform that:
- Collects biodiversity data from field staff via KoboToolbox mobile forms
- Validates and processes submitted data through a multi-stage workflow
- Provides analytics, reporting, and GIS visualization capabilities
- Supports role-based access control for 5 user types

**Key Technologies:**
- **Backend:** Django 5.2 + Django REST Framework
- **Database:** PostgreSQL 16 + PostGIS (spatial extension)
- **Cache/Queue:** Redis 6.0+
- **Task Queue:** Celery with Celery Beat
- **Frontend:** Django Templates + Bootstrap 5 + Leaflet.js
- **External Integration:** KoboToolbox API

---

## 2. Pre-Installation Checklist

### 2.1 System Requirements

#### Minimum Requirements
- **OS:** Ubuntu 20.04+ / Debian 11+ / Windows 10+ with Docker
- **CPU:** 2 cores
- **RAM:** 4GB (8GB recommended for production)
- **Storage:** 20GB free space (SSD recommended)
- **Network:** Static IP address or domain name

#### Recommended Production Requirements
- **CPU:** 4 cores
- **RAM:** 16GB
- **Storage:** 100GB SSD
- **Network:** Domain with SSL certificate

### 2.2 Required Access & Credentials

Prepare the following before starting:

- [ ] **Server Access:** SSH access with sudo privileges
- [ ] **Domain Name:** Your production domain (e.g., prism.yourorg.gov.ph)
- [ ] **Email Account:** SMTP credentials for system notifications
- [ ] **KoboToolbox Account:** API token from KoboToolbox (https://eu.kobotoolbox.org/)
- [ ] **SSL Certificate:** Let's Encrypt or purchased certificate
- [ ] **GitHub Access:** If cloning from private repository

### 2.3 Knowledge Prerequisites

This guide assumes familiarity with:
- Basic Linux command line operations
- Docker and Docker Compose (recommended approach)
- PostgreSQL database administration
- Django framework basics
- Git version control

---

## 3. Phase 1: Infrastructure Setup

### 3.1 Option A: Docker Installation (Recommended)

Docker provides the fastest and most reliable deployment method.

#### Install Docker & Docker Compose

**On Ubuntu/Debian:**
```bash
# Update package index
sudo apt update && sudo apt upgrade -y

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Add your user to docker group
sudo usermod -aG docker $USER

# Install Docker Compose
sudo apt install docker-compose-plugin

# Verify installation
docker --version
docker compose version
```

**On Windows:**
1. Download Docker Desktop from https://www.docker.com/products/docker-desktop/
2. Install and restart your computer
3. Enable WSL 2 backend if prompted

#### Clone Repository

```bash
# Navigate to your projects directory
cd /var/www  # Linux
cd C:\Projects  # Windows

# Clone the repository
git clone https://github.com/your-org/PRISM-Matutum.git
cd PRISM-Matutum
```

#### Configure Environment Variables

```bash
# Copy example environment file
cp .env.example .env

# Edit with your preferred editor
nano .env  # or vim, code, notepad++
```

**Critical Environment Variables to Configure:**

```env
# Django Configuration
DJANGO_SECRET_KEY=<generate-using-command-below>
DEBUG=False
ALLOWED_HOSTS=yourdomain.com,www.yourdomain.com,localhost
ENVIRONMENT=production

# Database Configuration
DB_NAME=prismdb
DB_USER=prismuser
DB_PASSWORD=<strong-password-here>
DB_HOST=db  # Docker service name
DB_PORT=5432

# Redis Configuration
REDIS_URL=redis://redis:6379/1

# Email Configuration (Gmail example)
EMAIL_HOST=smtp.gmail.com
EMAIL_PORT=587
EMAIL_USE_TLS=True
EMAIL_HOST_USER=your-email@gmail.com
EMAIL_HOST_PASSWORD=<app-password>
DEFAULT_FROM_EMAIL=PRISM Matutum <your-email@gmail.com>

# KoboToolbox Integration
KOBO_API_BASE_URL=https://kf.kobotoolbox.org/api/v2
KOBO_API_TOKEN=<your-kobo-api-token>

# Security Settings (for production)
SECURE_SSL_REDIRECT=True
SESSION_COOKIE_SECURE=True
CSRF_COOKIE_SECURE=True
SECURE_HSTS_SECONDS=31536000
```

**Generate Secret Key:**
```bash
python -c 'from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())'
```

#### Build and Start Containers

```bash
# Build the Docker images
docker compose build

# Start all services in detached mode
docker compose up -d

# Verify all containers are running
docker compose ps

# Expected output:
# NAME                STATUS
# prism-web          Up
# prism-db           Up (healthy)
# prism-redis        Up
# prism-celery       Up
# prism-celery-beat  Up
```

#### Check Logs

```bash
# View logs for all services
docker compose logs -f

# View logs for specific service
docker compose logs -f web
docker compose logs -f db
docker compose logs -f celery
```

### 3.2 Option B: Manual Installation (Linux)

If you prefer manual installation without Docker:

#### Install System Dependencies

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Python 3.10+
sudo apt install -y python3.10 python3.10-venv python3-pip python3-dev

# Install PostgreSQL 16 with PostGIS
sudo apt install -y postgresql-16 postgresql-16-postgis-3 postgresql-client-16

# Install Redis
sudo apt install -y redis-server

# Install Nginx
sudo apt install -y nginx

# Install Supervisor (for Celery)
sudo apt install -y supervisor

# Install build dependencies
sudo apt install -y libpq-dev build-essential libgdal-dev
```

#### Create Application User

```bash
# Create dedicated user for the application
sudo useradd -m -s /bin/bash prismapp
sudo usermod -aG www-data prismapp
```

#### Clone and Setup Application

```bash
# Switch to application user
sudo su - prismapp

# Clone repository
cd /home/prismapp
git clone https://github.com/your-org/PRISM-Matutum.git
cd PRISM-Matutum

# Create virtual environment
python3.10 -m venv venv
source venv/bin/activate

# Install Python dependencies
pip install --upgrade pip
pip install -r requirements.txt
```

---

## 4. Phase 2: Application Installation

### 4.1 Database Initialization

#### Run Migrations

**Docker Environment:**
```bash
# Run Django migrations
docker compose exec web python manage.py migrate

# Verify migration status
docker compose exec web python manage.py showmigrations
```

**Manual Installation:**
```bash
source venv/bin/activate
python manage.py migrate
```

#### Create Superuser

```bash
# Docker
docker compose exec web python manage.py createsuperuser

# Manual
python manage.py createsuperuser
```

**Follow the prompts:**
```
Username: admin
Email: admin@yourdomain.com
Password: <strong-password>
Password (again): <strong-password>
```

### 4.2 Collect Static Files

```bash
# Docker
docker compose exec web python manage.py collectstatic --noinput

# Manual
python manage.py collectstatic --noinput
```

### 4.3 Verify Installation

```bash
# Check Django is running
curl http://localhost:8000

# Or visit in browser
# http://localhost:8000
```

You should see the PRISM-Matutum login page.

---

## 5. Phase 3: Database Configuration

### 5.1 Access Database

**Docker Environment:**
```bash
# Access PostgreSQL shell
docker compose exec db psql -U prismuser -d prismdb
```

**Manual Installation:**
```bash
sudo su - postgres
psql -d prismdb
```

### 5.2 Verify PostGIS Extension

```sql
-- Check PostGIS is installed
SELECT PostGIS_version();

-- Should return something like: 3.3.2

-- Verify spatial reference systems
SELECT count(*) FROM spatial_ref_sys;

-- Should return 8500+ rows
```

### 5.3 Database Backup Configuration

```bash
# Create backup directory
mkdir -p /var/backups/prism

# Create backup script
cat > /usr/local/bin/prism-backup.sh << 'EOF'
#!/bin/bash
BACKUP_DIR="/var/backups/prism"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
DB_NAME="prismdb"
DB_USER="prismuser"

# For Docker
docker compose exec -T db pg_dump -U $DB_USER $DB_NAME | gzip > $BACKUP_DIR/prism_${TIMESTAMP}.sql.gz

# Keep only last 30 days
find $BACKUP_DIR -name "prism_*.sql.gz" -mtime +30 -delete

echo "Backup completed: prism_${TIMESTAMP}.sql.gz"
EOF

# Make executable
chmod +x /usr/local/bin/prism-backup.sh

# Test backup
/usr/local/bin/prism-backup.sh
```

### 5.4 Schedule Automated Backups

```bash
# Add to crontab
crontab -e

# Add this line (daily at 2 AM)
0 2 * * * /usr/local/bin/prism-backup.sh >> /var/log/prism-backup.log 2>&1
```

---

## 6. Phase 4: Initial Data Population

This is the most critical phase for getting your system production-ready. Follow these steps in order.

### 6.1 Initialize Choice Lists

Choice lists are used in KoboToolbox forms for dropdown menus.

```bash
# Docker
docker compose exec web python manage.py initialize_choice_lists

# Manual
python manage.py initialize_choice_lists
```

**What this creates:**
- Mode of Observation choices (Seen, Heard, Traces, etc.)
- Resource Use Type choices (Timber, Hunting, Fishing, etc.)
- Disturbance Type choices (Deforestation, Fire, Erosion, etc.)
- Severity Level choices (Low, Medium, High, Critical)
- Weather Condition choices (Sunny, Cloudy, Rain, etc.)

### 6.2 Populate Taxonomy Data

Taxonomy is the hierarchical classification of species (Kingdom → Phylum → Class → Order → Family → Genus → Species).

#### Create Initial Taxonomy Records

Access Django shell:
```bash
# Docker
docker compose exec web python manage.py shell

# Manual
python manage.py shell
```

**In the Django shell:**

```python
from species.models import Kingdom, Phylum, Class, Order, Family, Genus, Taxonomy

# Create Kingdoms
animalia = Kingdom.objects.create(
    kingdom_name='Animalia',
    description='Animal kingdom - all fauna species'
)

plantae = Kingdom.objects.create(
    kingdom_name='Plantae',
    description='Plant kingdom - all flora species'
)

# Create Phylum (Example for Animalia)
chordata = Phylum.objects.create(
    id_kingdom=animalia,
    phylum_name='Chordata',
    description='Animals with notochord/backbone'
)

# Create Class (Example)
mammalia = Class.objects.create(
    id_phylum=chordata,
    class_name='Mammalia',
    description='Mammals - warm-blooded vertebrates'
)

aves = Class.objects.create(
    id_phylum=chordata,
    class_name='Aves',
    description='Birds - feathered, winged, egg-laying vertebrates'
)

# Create Order (Example)
primates = Order.objects.create(
    id_class=mammalia,
    order_name='Primates',
    description='Primates - monkeys, apes, lemurs, humans'
)

# Create Family (Example)
cercopithecidae = Family.objects.create(
    id_order=primates,
    family_name='Cercopithecidae',
    description='Old World monkeys'
)

# Create Genus (Example)
macaca = Genus.objects.create(
    id_family=cercopithecidae,
    genus_name='Macaca',
    description='Macaques'
)

# Create complete Taxonomy record
macaca_fascicularis_taxonomy = Taxonomy.objects.create(
    id_kingdom=animalia,
    id_phylum=chordata,
    id_class=mammalia,
    id_order=primates,
    id_family=cercopithecidae,
    id_genus=macaca,
    species='fascicularis',
    taxonomy_notes='Long-tailed macaque, common in Southeast Asia'
)

print("Taxonomy created successfully!")
exit()
```

#### Bulk Import Taxonomy (Recommended for Production)

For a complete taxonomy, create a CSV file:

**taxonomy_import.csv:**
```csv
kingdom,phylum,class,order,family,genus,species,common_name,notes
Animalia,Chordata,Mammalia,Primates,Cercopithecidae,Macaca,fascicularis,Long-tailed Macaque,Common in SE Asia
Animalia,Chordata,Mammalia,Carnivora,Viverridae,Paradoxurus,hermaphroditus,Asian Palm Civet,Frugivorous species
Animalia,Chordata,Aves,Passeriformes,Dicruridae,Dicrurus,balicassius,Balicassiao,Philippine endemic bird
Plantae,Tracheophyta,Magnoliopsida,Dipterocarpales,Dipterocarpaceae,Shorea,contorta,White Lauan,Philippine mahogany
```

**Import script (create as `scripts/import_taxonomy.py`):**

```python
import csv
import os
import django

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'prism.settings')
django.setup()

from species.models import Kingdom, Phylum, Class, Order, Family, Genus, Taxonomy

def import_taxonomy(csv_file):
    with open(csv_file, 'r', encoding='utf-8') as f:
        reader = csv.DictReader(f)
        
        for row in reader:
            # Get or create Kingdom
            kingdom, _ = Kingdom.objects.get_or_create(
                kingdom_name=row['kingdom'],
                defaults={'description': f"{row['kingdom']} kingdom"}
            )
            
            # Get or create Phylum
            phylum, _ = Phylum.objects.get_or_create(
                id_kingdom=kingdom,
                phylum_name=row['phylum'],
                defaults={'description': f"{row['phylum']} phylum"}
            )
            
            # Get or create Class
            cls, _ = Class.objects.get_or_create(
                id_phylum=phylum,
                class_name=row['class'],
                defaults={'description': f"{row['class']} class"}
            )
            
            # Get or create Order
            order, _ = Order.objects.get_or_create(
                id_class=cls,
                order_name=row['order'],
                defaults={'description': f"{row['order']} order"}
            )
            
            # Get or create Family
            family, _ = Family.objects.get_or_create(
                id_order=order,
                family_name=row['family'],
                defaults={'description': f"{row['family']} family"}
            )
            
            # Get or create Genus
            genus, _ = Genus.objects.get_or_create(
                id_family=family,
                genus_name=row['genus'],
                defaults={'description': f"{row['genus']} genus"}
            )
            
            # Create Taxonomy
            taxonomy, created = Taxonomy.objects.get_or_create(
                id_kingdom=kingdom,
                id_phylum=phylum,
                id_class=cls,
                id_order=order,
                id_family=family,
                id_genus=genus,
                species=row['species'],
                defaults={'taxonomy_notes': row.get('notes', '')}
            )
            
            if created:
                print(f"✓ Created: {genus.genus_name} {row['species']}")
            else:
                print(f"○ Exists: {genus.genus_name} {row['species']}")

if __name__ == '__main__':
    import_taxonomy('taxonomy_import.csv')
    print("\nTaxonomy import completed!")
```

**Run the import:**
```bash
# Docker
docker compose exec web python scripts/import_taxonomy.py

# Manual
python scripts/import_taxonomy.py
```

### 6.3 Populate Conservation Status Data

```bash
docker compose exec web python manage.py shell
```

```python
from species.models import Endemicity, ThreatAssessment

# Create Endemicity levels
endemicity_data = [
    ('endemic_philippines', 'Endemic to Philippines', 'Found only in the Philippines'),
    ('endemic_mindanao', 'Endemic to Mindanao', 'Found only in Mindanao region'),
    ('endemic_matutum', 'Endemic to Mt. Matutum', 'Found only in Mt. Matutum area'),
    ('native', 'Native (not endemic)', 'Native to Philippines but not endemic'),
    ('introduced', 'Introduced/Alien', 'Non-native species introduced to the area'),
]

for code, name, desc in endemicity_data:
    Endemicity.objects.get_or_create(
        endemicity_name=name,
        defaults={'description': desc}
    )
    print(f"✓ Created: {name}")

# Create IUCN Threat Categories
threat_data = [
    ('EX', 'Extinct', 'No individuals remaining'),
    ('EW', 'Extinct in the Wild', 'Only in captivity'),
    ('CR', 'Critically Endangered', 'Extremely high risk of extinction'),
    ('EN', 'Endangered', 'High risk of extinction'),
    ('VU', 'Vulnerable', 'High risk of endangerment'),
    ('NT', 'Near Threatened', 'Close to qualifying for threatened'),
    ('LC', 'Least Concern', 'Low risk of extinction'),
    ('DD', 'Data Deficient', 'Inadequate information'),
    ('NE', 'Not Evaluated', 'Not yet assessed'),
]

for code, name, desc in threat_data:
    ThreatAssessment.objects.get_or_create(
        threat_category=name,
        defaults={
            'threat_description': desc,
            'iucn_code': code
        }
    )
    print(f"✓ Created: {name} ({code})")

print("\nConservation status data created successfully!")
exit()
```

### 6.4 Add Species to Checklist

#### Add Fauna Species

```bash
docker compose exec web python manage.py shell
```

```python
from species.models import (
    FaunaChecklist, Taxonomy, Endemicity, ThreatAssessment,
    SpeciesType, FaunaCommonName, SpeciesCommonName
)

# Get taxonomy (example: Macaca fascicularis)
taxonomy = Taxonomy.objects.get(
    id_genus__genus_name='Macaca',
    species='fascicularis'
)

# Get conservation status
endemicity = Endemicity.objects.get(endemicity_name='Native (not endemic)')
threat = ThreatAssessment.objects.get(iucn_code='LC')

# Get or create SpeciesType (auto-assigned based on taxonomy)
species_type, _ = SpeciesType.objects.get_or_create(
    type_name='Mammal',
    defaults={'description': 'Mammals'}
)

# Create Fauna Checklist entry
fauna = FaunaChecklist.objects.create(
    scientific_name='Macaca fascicularis',
    id_taxonomy=taxonomy,
    id_species_type=species_type,
    id_endemicity=endemicity,
    id_threat_category=threat,
    remarks='Common long-tailed macaque found in forest edges'
)

# Add common names
common_name_obj = SpeciesCommonName.objects.create(
    common_name='Long-tailed Macaque',
    language='English'
)
FaunaCommonName.objects.create(
    id_fauna_checklist=fauna,
    id_common_name=common_name_obj
)

common_name_local = SpeciesCommonName.objects.create(
    common_name='Ungoy',
    language='Filipino'
)
FaunaCommonName.objects.create(
    id_fauna_checklist=fauna,
    id_common_name=common_name_local
)

print(f"✓ Added species: {fauna.scientific_name}")
print(f"  Common names: Long-tailed Macaque (English), Ungoy (Filipino)")
exit()
```

#### Add Flora Species (Similar Pattern)

```python
from species.models import FloraChecklist, FloraCommonName

# Example: Add a tree species
taxonomy = Taxonomy.objects.get(
    id_genus__genus_name='Shorea',
    species='contorta'
)

species_type, _ = SpeciesType.objects.get_or_create(
    type_name='Tree',
    defaults={'description': 'Trees and large woody plants'}
)

flora = FloraChecklist.objects.create(
    scientific_name='Shorea contorta',
    id_taxonomy=taxonomy,
    id_species_type=species_type,
    id_endemicity=Endemicity.objects.get(endemicity_name='Endemic to Philippines'),
    id_threat_category=ThreatAssessment.objects.get(iucn_code='CR'),
    remarks='White Lauan, critically endangered Philippine mahogany'
)

common_name = SpeciesCommonName.objects.create(
    common_name='White Lauan',
    language='English'
)
FloraCommonName.objects.create(
    id_flora_checklist=flora,
    id_common_name=common_name
)

print(f"✓ Added species: {flora.scientific_name}")
exit()
```

#### Bulk Species Import (Recommended)

Create `scripts/import_species.py`:

```python
import csv
import os
import django

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'prism.settings')
django.setup()

from species.models import (
    FaunaChecklist, Taxonomy, Endemicity, ThreatAssessment,
    SpeciesType, FaunaCommonName, SpeciesCommonName
)

def import_fauna(csv_file):
    with open(csv_file, 'r', encoding='utf-8') as f:
        reader = csv.DictReader(f)
        
        for row in reader:
            try:
                # Get taxonomy
                taxonomy = Taxonomy.objects.get(
                    id_genus__genus_name=row['genus'],
                    species=row['species']
                )
                
                # Get conservation status
                endemicity = Endemicity.objects.get(
                    endemicity_name__icontains=row['endemicity']
                )
                threat = ThreatAssessment.objects.get(
                    iucn_code=row['iucn_code']
                )
                
                # Get species type
                species_type, _ = SpeciesType.objects.get_or_create(
                    type_name=row['species_type'],
                    defaults={'description': f"{row['species_type']} species"}
                )
                
                # Create or update fauna
                fauna, created = FaunaChecklist.objects.update_or_create(
                    scientific_name=f"{row['genus']} {row['species']}",
                    defaults={
                        'id_taxonomy': taxonomy,
                        'id_species_type': species_type,
                        'id_endemicity': endemicity,
                        'id_threat_category': threat,
                        'remarks': row.get('remarks', '')
                    }
                )
                
                # Add common names
                if row.get('common_name_en'):
                    common_name, _ = SpeciesCommonName.objects.get_or_create(
                        common_name=row['common_name_en'],
                        defaults={'language': 'English'}
                    )
                    FaunaCommonName.objects.get_or_create(
                        id_fauna_checklist=fauna,
                        id_common_name=common_name
                    )
                
                if row.get('common_name_local'):
                    common_name, _ = SpeciesCommonName.objects.get_or_create(
                        common_name=row['common_name_local'],
                        defaults={'language': 'Filipino'}
                    )
                    FaunaCommonName.objects.get_or_create(
                        id_fauna_checklist=fauna,
                        id_common_name=common_name
                    )
                
                status = "✓ Created" if created else "○ Updated"
                print(f"{status}: {fauna.scientific_name}")
                
            except Exception as e:
                print(f"✗ Error with {row.get('genus')} {row.get('species')}: {str(e)}")

if __name__ == '__main__':
    import_fauna('fauna_import.csv')
    print("\nFauna import completed!")
```

**fauna_import.csv example:**
```csv
genus,species,species_type,endemicity,iucn_code,common_name_en,common_name_local,remarks
Macaca,fascicularis,Mammal,Native,LC,Long-tailed Macaque,Ungoy,Common in forest edges
Paradoxurus,hermaphroditus,Mammal,Native,LC,Asian Palm Civet,Musang,Nocturnal frugivore
Dicrurus,balicassius,Bird,Endemic to Philippines,LC,Balicassiao,Balicassiao,Common endemic bird
```

**Run import:**
```bash
docker compose exec web python scripts/import_species.py
```

### 6.5 Populate Geographic Data

#### Administrative Divisions

```bash
docker compose exec web python manage.py shell
```

```python
from dashboard.models import Region, Province, Municipality, Barangay

# Create Region (Example: Region XII - SOCCSKSARGEN)
region = Region.objects.create(
    region_code='12',
    region_name='Region XII',
    region_description='SOCCSKSARGEN - South Cotabato, Cotabato, Sultan Kudarat, Sarangani, General Santos'
)

# Create Province
province = Province.objects.create(
    id_region=region,
    province_name='South Cotabato',
    province_description='Province in Region XII'
)

# Create Municipality
municipality = Municipality.objects.create(
    id_province=province,
    municipality_name='Tupi',
    municipality_description='Municipality near Mt. Matutum Protected Area'
)

# Create Barangays
barangays = [
    'Kablon',
    'Cebuano',
    'Miasong',
    'Palian',
]

for brgy_name in barangays:
    Barangay.objects.create(
        id_municipality=municipality,
        barangay_name=brgy_name,
        barangay_description=f'Barangay in Tupi, South Cotabato'
    )
    print(f"✓ Created barangay: {brgy_name}")

print("\nGeographic data created successfully!")
exit()
```

#### Monitoring Locations

```bash
docker compose exec web python manage.py shell
```

```python
from api.models import MonitoringLocation
from dashboard.models import Barangay
from django.contrib.gis.geos import Point

# Get barangay
barangay = Barangay.objects.get(barangay_name='Kablon')

# Create monitoring location
# Note: Coordinates for Mt. Matutum area (example)
location = MonitoringLocation.objects.create(
    location_name='Mt. Matutum Peak Trail',
    id_barangay=barangay,
    geom=Point(125.0789, 6.3628, srid=4326),  # Longitude, Latitude
    description='Main trail leading to Mt. Matutum peak',
    location_type='trail',
    is_active=True
)

print(f"✓ Created location: {location.location_name}")
print(f"  Coordinates: {location.geom.coords}")
exit()
```

---

## 7. Phase 5: User Management Setup

### 7.1 Create Organizational Structure

#### Create Agencies

```bash
docker compose exec web python manage.py shell
```

```python
from dashboard.models import Agency, Department

# Create main agency
denr = Agency.objects.create(
    agency_name='Department of Environment and Natural Resources',
    agency_acronym='DENR',
    description='DENR Region XII',
    is_active=True
)

# Create sub-agencies/offices
penro = Agency.objects.create(
    agency_name='Provincial Environment and Natural Resources Office',
    agency_acronym='PENRO',
    parent_agency=denr,
    description='PENR Office South Cotabato',
    is_active=True
)

pamb = Agency.objects.create(
    agency_name='Protected Area Management Board',
    agency_acronym='PAMB',
    parent_agency=denr,
    description='Mt. Matutum PAMB',
    is_active=True
)

# Create departments
dept_bio = Department.objects.create(
    department_name='Biodiversity Management',
    description='Biodiversity monitoring and research',
    id_agency=denr
)

dept_pa = Department.objects.create(
    department_name='Protected Area Management',
    description='PA management and enforcement',
    id_agency=penro
)

print("✓ Created agencies and departments")
exit()
```

### 7.2 Create User Accounts

#### Via Django Admin (Recommended for initial users)

1. Access Django admin: http://localhost:8000/admin/
2. Login with superuser credentials
3. Navigate to **Users** section
4. Click **Add User**
5. Fill in required fields:
   - Username
   - Email
   - Password
   - Role (select from dropdown)
   - Agency affiliation
   - Contact information
6. Set **Account Status** to "Approved"
7. Check **Is Active**
8. Save

#### Via Django Shell (Programmatic)

```bash
docker compose exec web python manage.py shell
```

```python
from users.models import User
from dashboard.models import Agency

# Get agency
denr = Agency.objects.get(agency_acronym='DENR')

# Create PA Staff user
pa_staff = User.objects.create_user(
    username='juan.cruz',
    email='juan.cruz@denr.gov.ph',
    password='TempPassword123!',
    first_name='Juan',
    last_name='Cruz',
    role='pa_staff',
    id_agency=denr,
    contact_number='+639171234567',
    account_status='approved',
    is_active=True
)

print(f"✓ Created PA Staff: {pa_staff.get_full_name()}")

# Create Validator user
validator = User.objects.create_user(
    username='maria.santos',
    email='maria.santos@denr.gov.ph',
    password='TempPassword123!',
    first_name='Maria',
    last_name='Santos',
    role='validator',
    id_agency=denr,
    contact_number='+639187654321',
    account_status='approved',
    is_active=True
)

print(f"✓ Created Validator: {validator.get_full_name()}")

# Create DENR Personnel
denr_user = User.objects.create_user(
    username='pedro.reyes',
    email='pedro.reyes@denr.gov.ph',
    password='TempPassword123!',
    first_name='Pedro',
    last_name='Reyes',
    role='denr',
    id_agency=denr,
    contact_number='+639199876543',
    account_status='approved',
    is_active=True
)

print(f"✓ Created DENR Personnel: {denr_user.get_full_name()}")

print("\n⚠️  Remember to ask users to change their passwords on first login!")
exit()
```

#### Bulk User Import

Create `scripts/import_users.py`:

```python
import csv
import os
import django

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'prism.settings')
django.setup()

from users.models import User
from dashboard.models import Agency

def import_users(csv_file):
    with open(csv_file, 'r', encoding='utf-8') as f:
        reader = csv.DictReader(f)
        
        for row in reader:
            try:
                # Get agency
                agency = Agency.objects.get(agency_acronym=row['agency'])
                
                # Create user
                user = User.objects.create_user(
                    username=row['username'],
                    email=row['email'],
                    password=row['temp_password'],
                    first_name=row['first_name'],
                    last_name=row['last_name'],
                    role=row['role'],
                    id_agency=agency,
                    contact_number=row.get('contact', ''),
                    account_status='approved',
                    is_active=True
                )
                
                print(f"✓ Created: {user.get_full_name()} ({user.role})")
                
            except Exception as e:
                print(f"✗ Error with {row.get('username')}: {str(e)}")

if __name__ == '__main__':
    import_users('users_import.csv')
    print("\nUser import completed!")
    print("⚠️  IMPORTANT: Email all users to change their temporary passwords!")
```

**users_import.csv:**
```csv
username,email,first_name,last_name,role,agency,contact,temp_password
juan.cruz,juan.cruz@denr.gov.ph,Juan,Cruz,pa_staff,DENR,+639171234567,TempPass123!
maria.santos,maria.santos@denr.gov.ph,Maria,Santos,validator,DENR,+639187654321,TempPass123!
pedro.reyes,pedro.reyes@denr.gov.ph,Pedro,Reyes,denr,PENRO,+639199876543,TempPass123!
```

### 7.3 Configure User Roles & Permissions

The system has 5 built-in roles with predefined permissions:

| Role | Code | Key Permissions |
|------|------|----------------|
| **System Admin** | `system_admin` | Full system access, user management, configuration |
| **Validator** | `validator` | Validate submissions, manage observations |
| **PA Staff** | `pa_staff` | Submit data, view own submissions |
| **DENR Personnel** | `denr` | View reports, analytics, export data |
| **Collaborator** | `collaborator` | View data, limited reporting |

**Verify permissions:**
```bash
docker compose exec web python manage.py shell
```

```python
from users.models import User

# Check user permissions
user = User.objects.get(username='juan.cruz')
print(f"User: {user.get_full_name()}")
print(f"Role: {user.role}")
print(f"Can submit data: {user.can_submit_data()}")
print(f"Can validate: {user.can_validate()}")
print(f"Is staff: {user.is_staff}")

# List all permissions
from dashboard.permissions import RolePermissionMatrix
print("\nRole Permissions:")
for role, perms in RolePermissionMatrix.items():
    print(f"\n{role}:")
    for perm in perms:
        print(f"  - {perm}")
exit()
```

---

## 8. Phase 6: KoboToolbox Integration

### 8.1 Create KoboToolbox Account

1. Visit https://www.kobotoolbox.org/
2. Click **Sign Up** (or use existing account)
3. Create a free account (or upgrade for production features)
4. Verify your email

### 8.2 Generate API Token

1. Login to KoboToolbox
2. Click on your profile icon (top right)
3. Select **Account Settings**
4. Go to **Security** tab
5. Under **API Token**, click **Generate New Token**
6. Copy the token (you won't see it again!)

### 8.3 Configure API Token in PRISM

**Update `.env` file:**
```env
KOBO_API_BASE_URL=https://eu.kobotoolbox.org/api/v2
KOBO_API_TOKEN=your_actual_token_here
```

**Restart containers to apply changes:**
```bash
docker compose down
docker compose up -d
```

### 8.4 Create Wildlife Observation Form in Kobo

1. In KoboToolbox, click **New Project**
2. Select **Build from scratch**
3. Name your form: "Wildlife Observation Form"

**Add these questions:**

| Question | Type | Name | Choice List |
|----------|------|------|-------------|
| Observer Name | Text | observer_name | - |
| Observation Date | Date | observation_date | - |
| Observation Time | Time | observation_time | - |
| Species Observed | Select One | species_name | (will be synced from Django) |
| Mode of Observation | Select One | mode_observation | mode_of_observation |
| Number of Individuals | Integer | individual_count | - |
| Location | GPS | observation_location | - |
| Barangay | Select One | barangay | (will be synced from Django) |
| Weather | Select One | weather_condition | weather_condition |
| Notes | Text | observation_notes | - |

4. Click **Deploy** when done

### 8.5 Link Kobo Form to Django

**Access Django Admin:**
1. Go to http://localhost:8000/admin/
2. Navigate to **API → Form Choice Mappings**
3. Click **Add Form Choice Mapping**

**Create mappings:**

**Mapping 1: Mode of Observation**
- **Asset UID**: (Copy from Kobo form URL - looks like `aXXXXXXXXXXXXXX`)
- **Question Name**: `mode_observation`
- **Choice List**: Select "mode_of_observation"
- **Auto Update**: ☑️ Checked

**Mapping 2: Weather Condition**
- **Asset UID**: (Same as above)
- **Question Name**: `weather_condition`
- **Choice List**: Select "weather_condition"
- **Auto Update**: ☑️ Checked

### 8.6 Sync Species Choices to Kobo

```bash
docker compose exec web python manage.py shell
```

```python
from api.models import ChoiceList, Choice
from species.models import FaunaChecklist

# Create species choice list
species_list, created = ChoiceList.objects.get_or_create(
    name='fauna_species',
    defaults={'description': 'Fauna species for field observations'}
)

# Add all fauna species as choices
for fauna in FaunaChecklist.objects.filter(id_threat_category__isnull=False):
    choice, created = Choice.objects.get_or_create(
        choice_list=species_list,
        name=fauna.scientific_name.replace(' ', '_').lower(),
        defaults={
            'label': f"{fauna.scientific_name} - {fauna.common_names.first().id_common_name.common_name if fauna.common_names.exists() else ''}",
            'order': fauna.id_fauna_checklist,
            'fauna_species': fauna
        }
    )
    if created:
        print(f"✓ Added: {fauna.scientific_name}")

print(f"\nTotal species choices: {species_list.choices.count()}")
exit()
```

**Create Form Mapping for Species:**
1. Go to Django Admin → Form Choice Mappings
2. Add new mapping:
   - Asset UID: (Your Kobo form ID)
   - Question Name: `species_name`
   - Choice List: fauna_species
   - Auto Update: ☑️

### 8.7 Push Choices to Kobo

```bash
# Docker
docker compose exec web python manage.py shell
```

```python
from api.form_services import KoboFormService

service = KoboFormService()

# Get your form's asset UID (from Kobo URL)
asset_uid = 'aXXXXXXXXXXXXXX'  # Replace with actual UID

# Push all choices to form
result = service.push_choices_to_form(asset_uid)

print(f"Status: {result['status']}")
print(f"Updated questions: {result['updated_questions']}")

exit()
```

### 8.8 Test Data Submission

1. Open KoboToolbox Collect app on mobile device
2. Download the form
3. Fill out a test observation
4. Submit the form

**Check submission in Django:**
```bash
docker compose exec web python manage.py shell
```

```python
from api.models import DataSubmission

# List recent submissions
submissions = DataSubmission.objects.all().order_by('-date_posted')[:5]

for sub in submissions:
    print(f"Submission ID: {sub.id_data_submission}")
    print(f"  Status: {sub.validation_status}")
    print(f"  Date: {sub.date_posted}")
    print(f"  Form: {sub.form_uid}")
    print()

exit()
```

### 8.9 Configure Automatic Sync

**Enable Celery Beat schedule:**

Edit `celery_config.py` or verify schedule exists:

```python
# celery_config.py
from celery.schedules import crontab

CELERYBEAT_SCHEDULE = {
    'sync-kobotoolbox-data': {
        'task': 'api.tasks.sync_kobotoolbox_data',
        'schedule': crontab(minute='*/30'),  # Every 30 minutes
    },
}
```

**Restart Celery Beat:**
```bash
docker compose restart celery-beat
docker compose logs -f celery-beat
```

**Manual sync command:**
```bash
# Force immediate sync
docker compose exec web python manage.py sync_kobotoolbox_data
```

---

## 9. Phase 7: Production Hardening

### 9.1 Security Configuration

#### Update Production Settings

Edit `prism/settings_production.py`:

```python
# Security settings
DEBUG = False
ALLOWED_HOSTS = ['yourdomain.com', 'www.yourdomain.com']

# HTTPS settings
SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SECURE_HSTS_SECONDS = 31536000  # 1 year
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True

# Additional security
SECURE_CONTENT_TYPE_NOSNIFF = True
SECURE_BROWSER_XSS_FILTER = True
X_FRAME_OPTIONS = 'DENY'

# Session security
SESSION_COOKIE_AGE = 7200  # 2 hours
SESSION_COOKIE_HTTPONLY = True
SESSION_SAVE_EVERY_REQUEST = True

# Password validation
AUTH_PASSWORD_VALIDATORS = [
    {'NAME': 'django.contrib.auth.password_validation.UserAttributeSimilarityValidator'},
    {'NAME': 'django.contrib.auth.password_validation.MinimumLengthValidator', 'OPTIONS': {'min_length': 12}},
    {'NAME': 'django.contrib.auth.password_validation.CommonPasswordValidator'},
    {'NAME': 'django.contrib.auth.password_validation.NumericPasswordValidator'},
]
```

#### Configure Nginx (Reverse Proxy)

Create `/etc/nginx/sites-available/prism`:

```nginx
upstream prism_backend {
    server localhost:8000;
}

server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;
    
    # Redirect HTTP to HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name yourdomain.com www.yourdomain.com;
    
    # SSL Configuration
    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    
    # Security headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options "DENY" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    
    # Client body size (for file uploads)
    client_max_body_size 100M;
    
    # Static files
    location /static/ {
        alias /var/www/PRISM-Matutum/staticfiles/;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }
    
    # Media files
    location /media/ {
        alias /var/www/PRISM-Matutum/media/;
        expires 7d;
    }
    
    # Proxy to Django
    location / {
        proxy_pass http://prism_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_redirect off;
    }
}
```

**Enable site:**
```bash
sudo ln -s /etc/nginx/sites-available/prism /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

#### Setup SSL Certificate (Let's Encrypt)

```bash
# Install Certbot
sudo apt install -y certbot python3-certbot-nginx

# Obtain certificate
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com

# Test auto-renewal
sudo certbot renew --dry-run
```

### 9.2 Configure Firewall

```bash
# Install UFW
sudo apt install -y ufw

# Allow SSH (IMPORTANT: Do this first!)
sudo ufw allow 22/tcp

# Allow HTTP and HTTPS
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Enable firewall
sudo ufw enable

# Check status
sudo ufw status
```

### 9.3 Configure Automatic Updates

```bash
# Install unattended-upgrades
sudo apt install -y unattended-upgrades

# Enable automatic security updates
sudo dpkg-reconfigure -plow unattended-upgrades
```

### 9.4 Setup Log Rotation

Create `/etc/logrotate.d/prism`:

```
/var/www/PRISM-Matutum/logs/*.log {
    daily
    missingok
    rotate 14
    compress
    delaycompress
    notifempty
    create 0640 www-data www-data
    sharedscripts
}
```

---

## 10. Phase 8: Monitoring & Maintenance

### 10.1 Health Check Endpoint

**Test system health:**
```bash
curl http://localhost:8000/health/

# Expected response:
{
  "status": "healthy",
  "database": "connected",
  "redis": "connected",
  "celery": "running"
}
```

### 10.2 Setup Monitoring (Optional but Recommended)

#### Prometheus + Grafana (Docker)

Add to `docker-compose.yml`:

```yaml
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    ports:
      - "9090:9090"
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
  
  grafana:
    image: grafana/grafana:latest
    volumes:
      - grafana_data:/var/lib/grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin123
    depends_on:
      - prometheus

volumes:
  prometheus_data:
  grafana_data:
```

**Create `prometheus.yml`:**
```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prism'
    static_configs:
      - targets: ['web:8000']
```

### 10.3 Email Alerts Configuration

**Configure in `.env`:**
```env
# Email alerts
ALERT_EMAIL_RECIPIENTS=admin@yourdomain.com,manager@yourdomain.com
EMAIL_BACKEND=django.core.mail.backends.smtp.EmailBackend
EMAIL_HOST=smtp.gmail.com
EMAIL_PORT=587
EMAIL_USE_TLS=True
EMAIL_HOST_USER=alerts@yourdomain.com
EMAIL_HOST_PASSWORD=<app-password>
```

### 10.4 Scheduled Maintenance Tasks

**Create maintenance script `scripts/maintenance.sh`:**

```bash
#!/bin/bash

echo "=== PRISM-Matutum Maintenance Script ==="
echo "Started: $(date)"

# Backup database
echo "Creating database backup..."
docker compose exec -T db pg_dump -U prismuser prismdb | gzip > /var/backups/prism/prism_$(date +%Y%m%d_%H%M%S).sql.gz

# Clean old sessions
echo "Cleaning expired sessions..."
docker compose exec web python manage.py clearsessions

# Check for pending migrations
echo "Checking migrations..."
docker compose exec web python manage.py showmigrations | grep "\[ \]"

# Sync Kobo data
echo "Syncing KoboToolbox data..."
docker compose exec web python manage.py sync_kobotoolbox_data

# Collect static files
echo "Collecting static files..."
docker compose exec web python manage.py collectstatic --noinput

# Optimize database
echo "Optimizing database..."
docker compose exec db psql -U prismuser -d prismdb -c "VACUUM ANALYZE;"

echo "Maintenance completed: $(date)"
```

**Make executable and schedule:**
```bash
chmod +x scripts/maintenance.sh

# Add to crontab (weekly maintenance on Sundays at 2 AM)
crontab -e

# Add:
0 2 * * 0 /var/www/PRISM-Matutum/scripts/maintenance.sh >> /var/log/prism-maintenance.log 2>&1
```

### 10.5 Monitoring Dashboard

**Access system metrics:**
1. Django Admin: http://yourdomain.com/admin/
2. System Health: http://yourdomain.com/health/
3. Celery Flower (if installed): http://yourdomain.com:5555/

**Key metrics to monitor:**
- [ ] Database size and growth rate
- [ ] Number of pending validations
- [ ] Celery task queue length
- [ ] API response times
- [ ] Failed login attempts
- [ ] Kobo sync success rate
- [ ] Disk space utilization
- [ ] Memory usage

---

## 11. Troubleshooting

### 11.1 Common Issues

#### Database Connection Errors

**Problem:** `django.db.utils.OperationalError: could not connect to server`

**Solution:**
```bash
# Check if PostgreSQL is running
docker compose ps db

# Check database logs
docker compose logs db

# Verify connection settings in .env
docker compose exec db psql -U prismuser -d prismdb -c "SELECT version();"
```

#### Celery Not Processing Tasks

**Problem:** Tasks stuck in queue, not processing

**Solution:**
```bash
# Check Celery worker status
docker compose logs celery

# Restart Celery
docker compose restart celery celery-beat

# Check Redis connection
docker compose exec redis redis-cli ping
# Should return: PONG
```

#### KoboToolbox Sync Failing

**Problem:** `403 Forbidden` or `Authentication failed`

**Solution:**
```bash
# Verify API token
docker compose exec web python manage.py shell

# In shell:
from django.conf import settings
print(settings.KOBO_API_TOKEN)

# Test API connection
import requests
headers = {'Authorization': f'Token {settings.KOBO_API_TOKEN}'}
response = requests.get('https://kf.kobotoolbox.org/api/v2/assets/', headers=headers)
print(response.status_code, response.json())
```

#### Static Files Not Loading

**Problem:** CSS/JS not loading in production

**Solution:**
```bash
# Collect static files
docker compose exec web python manage.py collectstatic --noinput

# Check static files directory
docker compose exec web ls -la staticfiles/

# Verify Nginx configuration
sudo nginx -t
sudo systemctl reload nginx
```

#### Out of Disk Space

**Problem:** `No space left on device`

**Solution:**
```bash
# Check disk usage
df -h

# Clean Docker images/containers
docker system prune -a

# Clean old backups (keep last 30 days)
find /var/backups/prism -name "prism_*.sql.gz" -mtime +30 -delete

# Check large log files
du -sh /var/log/*
sudo truncate -s 0 /var/log/nginx/error.log
```

### 11.2 Debug Mode

**Enable debugging (DEVELOPMENT ONLY):**

Edit `.env`:
```env
DEBUG=True
LOG_LEVEL=DEBUG
```

Restart containers:
```bash
docker compose down
docker compose up -d
docker compose logs -f web
```

**⚠️ NEVER enable DEBUG in production!**

### 11.3 Get Support

- **Documentation:** `/docs` folder in repository
- **GitHub Issues:** https://github.com/your-org/PRISM-Matutum/issues
- **Email:** support@yourdomain.com

---

## 12. Quick Reference Commands

### Docker Commands

```bash
# Start all services
docker compose up -d

# Stop all services
docker compose down

# Restart specific service
docker compose restart web

# View logs
docker compose logs -f web

# Execute command in container
docker compose exec web python manage.py <command>

# Access Django shell
docker compose exec web python manage.py shell

# Access database
docker compose exec db psql -U prismuser -d prismdb

# Clean system
docker system prune -a
```

### Django Management Commands

```bash
# Database
python manage.py migrate
python manage.py makemigrations
python manage.py showmigrations

# Users
python manage.py createsuperuser
python manage.py changepassword <username>

# Data
python manage.py initialize_choice_lists
python manage.py sync_kobotoolbox_data
python manage.py loaddata <fixture>
python manage.py dumpdata <app> > data.json

# Static files
python manage.py collectstatic --noinput

# Shell
python manage.py shell
python manage.py dbshell

# Server
python manage.py runserver
```

### Backup & Restore

```bash
# Backup database
docker compose exec -T db pg_dump -U prismuser prismdb > backup.sql

# Restore database
docker compose exec -T db psql -U prismuser -d prismdb < backup.sql

# Backup with compression
docker compose exec -T db pg_dump -U prismuser prismdb | gzip > backup_$(date +%Y%m%d).sql.gz

# Restore from compressed backup
gunzip < backup_20260108.sql.gz | docker compose exec -T db psql -U prismuser -d prismdb
```

### System Checks

```bash
# Check system health
curl http://localhost:8000/health/

# Check database
docker compose exec db psql -U prismuser -d prismdb -c "SELECT count(*) FROM auth_user;"

# Check Redis
docker compose exec redis redis-cli ping

# Check Celery
docker compose exec web celery -A prism inspect active

# Check disk space
df -h

# Check memory
free -h

# Check Docker resources
docker stats
```

---

## ✅ Post-Deployment Checklist

- [ ] All Docker containers running (`docker compose ps`)
- [ ] Database migrations applied (`python manage.py showmigrations`)
- [ ] Superuser account created
- [ ] Static files collected
- [ ] Choice lists initialized
- [ ] Taxonomy data populated
- [ ] At least 10 species added to checklist
- [ ] Geographic data (regions, provinces, municipalities, barangays) populated
- [ ] Monitoring locations created
- [ ] Agencies and departments created
- [ ] User accounts created for each role
- [ ] KoboToolbox API token configured
- [ ] At least one form created in Kobo
- [ ] Form choice mappings configured
- [ ] Test data submission from Kobo to Django successful
- [ ] Celery workers running
- [ ] Celery Beat schedule active
- [ ] Automatic Kobo sync working (check every 30 min)
- [ ] Email notifications working
- [ ] SSL certificate installed (production only)
- [ ] Nginx reverse proxy configured (production only)
- [ ] Firewall rules applied (production only)
- [ ] Backup cron jobs scheduled
- [ ] Log rotation configured
- [ ] Health check endpoint responding
- [ ] All user roles tested with correct permissions
- [ ] Documentation reviewed by team

---

## 🎉 Congratulations!

Your PRISM-Matutum system is now fully deployed and operational!

**Next Steps:**
1. Train field staff on KoboToolbox Collect app
2. Train validators on the validation workflow
3. Schedule regular monitoring and backups
4. Plan for data analysis and reporting workshops
5. Review and update species checklists periodically

**For ongoing support, refer to:**
- User Guides: `/docs/user-guides/`
- Feature Documentation: `/docs/features/`
- Architecture Docs: `/docs/architecture/`

---

**Document Version:** 1.0  
**Last Updated:** January 8, 2026  
**Maintained by:** PRISM-Matutum Development Team
