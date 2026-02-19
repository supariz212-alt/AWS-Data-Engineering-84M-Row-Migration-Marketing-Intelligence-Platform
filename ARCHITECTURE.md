# AWS Data Engineering Solution Architecture
## 84M Record Migration & Marketing Intelligence Platform

---

## A. EXECUTIVE SUMMARY

This document outlines the complete technical architecture for migrating 84 million records from fragmented data silos (legacy MySQL, SQLite, CSV files) into a unified AWS-based "Source of Truth" with automated deduplication, compliance scrubbing, and a high-performance operations dashboard.

**Key Metrics:**
- **Data Volume:** 84M rows (68M legacy + 15M campaign CSV)
- **Target Performance:** Sub-1.5 second query response time
- **Availability:** 99.9% uptime SLA
- **Data Preservation:** 100% (zero data loss tolerance)

---

## B. TARGET ARCHITECTURE

### System Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         AWS CLOUD (VPC)                                  │
│                                                                           │
│  ┌────────────────────────── DATA SOURCES ──────────────────────────┐  │
│  │                                                                    │  │
│  │  • Legacy MySQL (68M rows)                                        │  │
│  │  • SQLite Databases                                               │  │
│  │  • Campaign CSV Files (15M rows)                                  │  │
│  │  • Pipedrive CRM (API)                                            │  │
│  │  • RingCentral Call Logs (API)                                    │  │
│  │  • Ongoing CSV/SQL Uploads                                        │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                  │                                        │
│                                  ▼                                        │
│  ┌────────────────────────── INGESTION LAYER ───────────────────────┐  │
│  │                                                                    │  │
│  │  ┌──────────────────┐    ┌──────────────────┐                   │  │
│  │  │  S3 Landing Zone  │───▶│  Lambda Trigger  │                   │  │
│  │  │  (Raw Files)      │    │  (Start Job)     │                   │  │
│  │  └──────────────────┘    └──────────────────┘                   │  │
│  │           │                        │                              │  │
│  │           ▼                        ▼                              │  │
│  │  ┌──────────────────────────────────────────────┐               │  │
│  │  │  AWS Glue ETL Jobs (Python/PySpark)          │               │  │
│  │  │  • File Validation & Schema Checks           │               │  │
│  │  │  • Data Type Conversion                       │               │  │
│  │  │  • Initial Cleaning (phone, names)           │               │  │
│  │  │  • Checksum Generation                        │               │  │
│  │  └──────────────────────────────────────────────┘               │  │
│  │           │                                                       │  │
│  │           ▼                                                       │  │
│  │  ┌──────────────────────────────────────────────┐               │  │
│  │  │  RDS Aurora MySQL (Staging Tables)           │               │  │
│  │  │  • staging_raw_import                         │               │  │
│  │  │  • import_metadata (versioning)               │               │  │
│  │  │  • import_validation_log                      │               │  │
│  │  └──────────────────────────────────────────────┘               │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                  │                                        │
│                                  ▼                                        │
│  ┌────────────────────────── PROCESSING LAYER ──────────────────────┐  │
│  │                                                                    │  │
│  │  ┌──────────────────────────────────────────────┐               │  │
│  │  │  AWS Glue / ECS (Processing Engine)          │               │  │
│  │  │                                                │               │  │
│  │  │  1. ATOMIC UPSERT ENGINE                      │               │  │
│  │  │     • Match Key Generation (email, phone)     │               │  │
│  │  │     • INSERT or UPDATE logic                  │               │  │
│  │  │     • Conflict Resolution                     │               │  │
│  │  │                                                │               │  │
│  │  │  2. DATA HYGIENE ENGINE                       │               │  │
│  │  │     • Phone Standardization (E.164)          │               │  │
│  │  │     • Name Parsing (First/Last/Middle)        │               │  │
│  │  │     • Address Normalization (USPS)           │               │  │
│  │  │                                                │               │  │
│  │  │  3. DEDUPLICATION ENGINE (Golden Record)      │               │  │
│  │  │     • Fuzzy Matching (Levenshtein Distance)  │               │  │
│  │  │     • Business Rules Application             │               │  │
│  │  │     • is_original vs is_duplicate tagging    │               │  │
│  │  │                                                │               │  │
│  │  │  4. COMPLIANCE ENGINE                         │               │  │
│  │  │     • DNC Registry Scrubbing                  │               │  │
│  │  │     • Opt-Out List Matching                   │               │  │
│  │  │     • Frequency Caps Enforcement              │               │  │
│  │  └──────────────────────────────────────────────┘               │  │
│  │           │                                                       │  │
│  │           ▼                                                       │  │
│  │  ┌──────────────────────────────────────────────┐               │  │
│  │  │  RDS Aurora MySQL (Canonical Tables)         │               │  │
│  │  │  • persons (deduplicated master)              │               │  │
│  │  │  • person_duplicates (linkage)                │               │  │
│  │  │  • addresses                                   │               │  │
│  │  │  • phone_numbers                               │               │  │
│  │  │  • email_addresses                             │               │  │
│  │  │  • campaigns                                   │               │  │
│  │  │  • touchpoints (mail/sms/call history)        │               │  │
│  │  │  • dnc_registry                                │               │  │
│  │  │  • opt_outs                                    │               │  │
│  │  └──────────────────────────────────────────────┘               │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                  │                                        │
│                                  ▼                                        │
│  ┌────────────────────────── APPLICATION LAYER ─────────────────────┐  │
│  │                                                                    │  │
│  │  ┌──────────────────┐    ┌──────────────────┐                   │  │
│  │  │  ElastiCache      │    │  CloudFront      │                   │  │
│  │  │  (Redis)          │    │  (CDN)           │                   │  │
│  │  │  Query Caching    │    │  Static Assets   │                   │  │
│  │  └──────────────────┘    └──────────────────┘                   │  │
│  │           │                        │                              │  │
│  │           ▼                        ▼                              │  │
│  │  ┌──────────────────────────────────────────────┐               │  │
│  │  │  ECS Fargate (API Layer)                      │               │  │
│  │  │  • FastAPI / Node.js                          │               │  │
│  │  │  • RESTful endpoints                          │               │  │
│  │  │  • WebSocket for real-time updates           │               │  │
│  │  └──────────────────────────────────────────────┘               │  │
│  │           │                                                       │  │
│  │           ▼                                                       │  │
│  │  ┌──────────────────────────────────────────────┐               │  │
│  │  │  Operations Dashboard (Web UI)                │               │  │
│  │  │  • Next.js / React                            │               │  │
│  │  │  • Hosted on S3 + CloudFront                  │               │  │
│  │  │                                                │               │  │
│  │  │  Features:                                     │               │  │
│  │  │  • Upload & Track Imports                     │               │  │
│  │  │  • Priority List Filtering                    │               │  │
│  │  │  • Lead Lifecycle Timeline                    │               │  │
│  │  │  • Compliance Risk Dashboard                  │               │  │
│  │  │  • Audit Log Viewer                           │               │  │
│  │  └──────────────────────────────────────────────┘               │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                  │                                        │
│                                  ▼                                        │
│  ┌────────────────────────── INTEGRATION LAYER ─────────────────────┐  │
│  │                                                                    │  │
│  │  ┌──────────────────┐    ┌──────────────────┐                   │  │
│  │  │  Pipedrive CRM   │◀──▶│  Sync Service    │                   │  │
│  │  │  (API)           │    │  (Bi-directional)│                   │  │
│  │  └──────────────────┘    └──────────────────┘                   │  │
│  │                                                                    │  │
│  │  ┌──────────────────┐    ┌──────────────────┐                   │  │
│  │  │  RingCentral     │───▶│  Ingestion Job   │                   │  │
│  │  │  Call Logs (API) │    │  (Scheduled)     │                   │  │
│  │  └──────────────────┘    └──────────────────┘                   │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                           │
│  ┌────────────────────────── OBSERVABILITY LAYER ───────────────────┐  │
│  │                                                                    │  │
│  │  • CloudWatch Logs & Metrics                                      │  │
│  │  • CloudWatch Alarms (Error Rate, Query Latency)                │  │
│  │  • X-Ray Tracing (Performance Bottlenecks)                       │  │
│  │  • SNS Notifications (Job Failures)                               │  │
│  │  • Kibana Dashboard (Log Analysis)                                │  │
│  └────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

### AWS Services Selected

| Service | Purpose | Justification |
|---------|---------|---------------|
| **RDS Aurora MySQL** | Primary database | MySQL compatibility + Auto-scaling + Read replicas for performance |
| **S3** | File landing zone | Durable storage, versioning, lifecycle policies |
| **AWS Glue** | ETL orchestration | Serverless, scales to 84M rows, built-in data catalog |
| **ECS Fargate** | API hosting | Containerized, auto-scaling, no server management |
| **ElastiCache (Redis)** | Query caching | Sub-second response times, reduces DB load |
| **Lambda** | Event triggers | S3 upload triggers, lightweight automation |
| **CloudWatch** | Monitoring | Centralized logs, metrics, alarms |
| **Secrets Manager** | Credential storage | Encrypted API keys, DB passwords, rotation |
| **VPC** | Network isolation | Private subnets, security groups, NAT gateway |
| **CloudFront** | CDN | Dashboard static assets, reduced latency |
| **EventBridge** | Scheduled jobs | Pipedrive sync, RingCentral ingestion |

---

## C. DATABASE SCHEMA

### Entity Relationship Model

```
┌─────────────────┐
│    persons      │ ◀───── Golden Record Master Table
├─────────────────┤
│ id (PK)         │
│ first_name      │
│ last_name       │
│ middle_name     │
│ is_original     │ ← TRUE for golden record
│ created_at      │
│ updated_at      │
└─────────────────┘
        │
        │ 1:N
        ▼
┌───────────────────┐
│ person_duplicates │ ◀───── Tracks which records are dupes
├───────────────────┤
│ id (PK)           │
│ original_id (FK)  │ ─────▶ persons.id
│ duplicate_id (FK) │ ─────▶ persons.id
│ match_score       │
│ match_reason      │
│ merged_at         │
└───────────────────┘

┌─────────────────┐
│ phone_numbers   │
├─────────────────┤
│ id (PK)         │
│ person_id (FK)  │ ─────▶ persons.id
│ phone_raw       │
│ phone_e164      │ ← Standardized format
│ phone_type      │ ← mobile, landline, voip
│ is_primary      │
│ dnc_status      │
└─────────────────┘

┌─────────────────┐
│ email_addresses │
├─────────────────┤
│ id (PK)         │
│ person_id (FK)  │ ─────▶ persons.id
│ email           │
│ is_primary      │
│ is_verified     │
│ opt_out_status  │
└─────────────────┘

┌─────────────────┐
│   addresses     │
├─────────────────┤
│ id (PK)         │
│ person_id (FK)  │ ─────▶ persons.id
│ street          │
│ city            │
│ state           │
│ zip             │
│ county          │
│ is_primary      │
└─────────────────┘

┌─────────────────┐
│   campaigns     │
├─────────────────┤
│ id (PK)         │
│ name            │
│ type            │ ← mail, sms, call
│ start_date      │
│ end_date        │
└─────────────────┘

┌─────────────────┐
│  touchpoints    │ ◀───── Life of a Lead
├─────────────────┤
│ id (PK)         │
│ person_id (FK)  │ ─────▶ persons.id
│ campaign_id (FK)│ ─────▶ campaigns.id
│ touchpoint_type │ ← mail, sms, call
│ touchpoint_date │
│ status          │ ← sent, delivered, failed
│ metadata        │ ← JSON (call duration, etc.)
└─────────────────┘

┌─────────────────┐
│ import_metadata │ ◀───── Versioning & Audit
├─────────────────┤
│ id (PK)         │
│ filename        │
│ source_type     │
│ import_date     │
│ row_count       │
│ success_count   │
│ error_count     │
│ checksum        │
│ s3_location     │
│ status          │
└─────────────────┘

┌─────────────────┐
│ dnc_registry    │
├─────────────────┤
│ id (PK)         │
│ phone_e164      │
│ listing_date    │
│ source          │
└─────────────────┘

┌─────────────────┐
│   opt_outs      │
├─────────────────┤
│ id (PK)         │
│ person_id (FK)  │
│ channel         │ ← sms, email, call
│ opted_out_at    │
│ reason          │
└─────────────────┘
```

---

## D. INGESTION ENGINE

### D1. Batch Import Flow

```
┌──────────────┐
│ User uploads │
│ CSV/SQL file │
└──────┬───────┘
       │
       ▼
┌──────────────────────┐
│ S3 Landing Zone      │
│ /raw-uploads/        │
│ YYYY/MM/DD/HH/       │
│ filename_timestamp   │
└──────┬───────────────┘
       │ Trigger
       ▼
┌──────────────────────┐
│ Lambda: validate     │
│ • File exists        │
│ • Size < 5GB         │
│ • Format: CSV/SQL    │
│ • Generate checksum  │
└──────┬───────────────┘
       │ Success
       ▼
┌──────────────────────┐
│ Create import_metadata│
│ record (status=PENDING)│
└──────┬───────────────┘
       │
       ▼
┌──────────────────────┐
│ Glue Job: ingest     │
│ • Read from S3       │
│ • Schema validation  │
│ • Type conversion    │
│ • Load to staging    │
└──────┬───────────────┘
       │
       ▼
┌──────────────────────┐
│ staging_raw_import   │
│ (All columns + meta) │
└──────┬───────────────┘
       │
       ▼
┌──────────────────────┐
│ Glue Job: process    │
│ • Data cleaning      │
│ • Deduplication      │
│ • Atomic upsert      │
└──────┬───────────────┘
       │
       ▼
┌──────────────────────┐
│ Canonical tables     │
│ (persons, phones...) │
└──────┬───────────────┘
       │
       ▼
┌──────────────────────┐
│ Update import_metadata│
│ (status=COMPLETE)    │
│ • row_count          │
│ • success_count      │
│ • error_count        │
│ Generate checklist   │
└──────────────────────┘
```

### D2. File Versioning Model

Every import is tracked with complete metadata:

```json
{
  "import_id": "imp_2026_02_19_001",
  "filename": "campaign_leads_q1_2026.csv",
  "s3_location": "s3://cm-data-lake/raw-uploads/2026/02/19/14/campaign_leads_q1_2026_20260219_140523.csv",
  "uploaded_by": "user@communityminerals.com",
  "uploaded_at": "2026-02-19T14:05:23Z",
  "file_size_bytes": 524288000,
  "row_count": 150000,
  "checksum_sha256": "a3f5...",
  "schema_version": "v2",
  "status": "COMPLETE",
  "processing_duration_seconds": 245,
  "success_count": 149850,
  "error_count": 150,
  "error_log_s3": "s3://cm-data-lake/error-logs/imp_2026_02_19_001_errors.csv"
}
```

### D3. Atomic Upsert Logic

**Match Keys:** `email` OR `phone_e164` OR (`first_name` + `last_name` + `zip`)

**SQL Pattern:**
```sql
-- Step 1: Identify existing records
CREATE TEMPORARY TABLE matched_records AS
SELECT 
    s.staging_id,
    p.id AS existing_person_id,
    CASE
        WHEN s.email = p.email THEN 'email'
        WHEN s.phone_e164 = ph.phone_e164 THEN 'phone'
        WHEN (s.first_name = p.first_name 
              AND s.last_name = p.last_name 
              AND s.zip = a.zip) THEN 'name_zip'
        ELSE NULL
    END AS match_type
FROM staging_raw_import s
LEFT JOIN persons p ON s.email = p.email
LEFT JOIN phone_numbers ph ON s.phone_e164 = ph.phone_e164
LEFT JOIN addresses a ON (s.first_name = p.first_name 
                          AND s.last_name = p.last_name 
                          AND s.zip = a.zip);

-- Step 2: UPDATE existing records (only if source data is newer)
UPDATE persons p
INNER JOIN matched_records m ON p.id = m.existing_person_id
INNER JOIN staging_raw_import s ON m.staging_id = s.staging_id
SET 
    p.first_name = COALESCE(s.first_name, p.first_name),
    p.last_name = COALESCE(s.last_name, p.last_name),
    p.updated_at = NOW(),
    p.last_import_id = s.import_id
WHERE s.source_date > p.updated_at;  -- Only update if newer

-- Step 3: INSERT new records
INSERT INTO persons (first_name, last_name, email, is_original, created_at, import_id)
SELECT 
    s.first_name,
    s.last_name,
    s.email,
    TRUE,  -- Initially marked as original
    NOW(),
    s.import_id
FROM staging_raw_import s
LEFT JOIN matched_records m ON s.staging_id = m.staging_id
WHERE m.existing_person_id IS NULL;

-- Step 4: Log the upsert operation
INSERT INTO import_audit_log (import_id, operation, record_count, timestamp)
VALUES 
    (?, 'UPDATE', @update_count, NOW()),
    (?, 'INSERT', @insert_count, NOW());
```

### D4. Validation Checklist Output

After each import, generate:

**Success Report (CSV):**
```csv
import_id,filename,total_rows,inserted,updated,skipped,errors,duration_sec,status
imp_001,leads.csv,150000,120000,25000,4850,150,245,COMPLETE
```

**Error Report (CSV):**
```csv
row_number,error_type,field_name,field_value,error_message
1523,VALIDATION,email,invalid@@email,"Invalid email format"
2891,MISSING_REQUIRED,phone,NULL,"Phone number is required"
5632,DUPLICATE,email,john@example.com,"Duplicate email in batch"
```

**Validation Checklist (JSON):**
```json
{
  "import_id": "imp_001",
  "checks": [
    {"check": "File readable", "status": "PASS"},
    {"check": "Schema matches expected", "status": "PASS"},
    {"check": "Required fields present", "status": "PASS"},
    {"check": "Row count matches file", "status": "PASS", "expected": 150000, "actual": 150000},
    {"check": "Duplicate rows in batch", "status": "WARN", "count": 47},
    {"check": "Invalid emails", "status": "FAIL", "count": 123},
    {"check": "Data types valid", "status": "PASS"}
  ],
  "overall_status": "COMPLETE_WITH_ERRORS"
}
```

### D5. Error Handling & Replay Strategy

**Error Categories:**
1. **Validation Errors:** Row skipped, logged, continue processing
2. **Database Errors:** Entire batch rolled back, job marked FAILED
3. **Network Errors:** Retry 3 times with exponential backoff

**Replay Mechanism:**
```bash
# Replay a failed import
python replay_import.py --import-id imp_001 --mode full

# Replay only failed rows
python replay_import.py --import-id imp_001 --mode errors-only
```

**Idempotency Guarantee:**
- Each import has unique `import_id`
- Re-running same import produces same result
- Checksums prevent duplicate file processing
- Upsert logic handles re-import of same data

---

## E. ETL + GOLDEN RECORD DEDUPLICATION

### E1. Standardization Rules

**Phone Numbers:**
```python
def standardize_phone(phone_raw):
    """
    Convert to E.164 format: +1AAABBBCCCC
    """
    # Remove all non-digits
    digits = re.sub(r'\D', '', phone_raw)
    
    # Add country code if missing (assume US)
    if len(digits) == 10:
        digits = '1' + digits
    
    # Format to E.164
    if len(digits) == 11 and digits[0] == '1':
        return f'+{digits}'
    
    return None  # Invalid phone

# Examples:
# (512) 555-1234 → +15125551234
# 512-555-1234 → +15125551234
# 5125551234 → +15125551234
```

**Name Parsing:**
```python
def parse_name(full_name):
    """
    Parse "FirstName MiddleName LastName" or "LastName, FirstName"
    """
    # Handle "LastName, FirstName" format
    if ',' in full_name:
        parts = full_name.split(',')
        last_name = parts[0].strip()
        first_middle = parts[1].strip().split()
        first_name = first_middle[0] if first_middle else ''
        middle_name = ' '.join(first_middle[1:]) if len(first_middle) > 1 else ''
        return first_name, middle_name, last_name
    
    # Handle "FirstName MiddleName LastName" format
    parts = full_name.strip().split()
    if len(parts) == 1:
        return parts[0], '', ''
    elif len(parts) == 2:
        return parts[0], '', parts[1]
    else:
        return parts[0], ' '.join(parts[1:-1]), parts[-1]

# Examples:
# "John Michael Smith" → ("John", "Michael", "Smith")
# "Smith, John" → ("John", "", "Smith")
# "Mary Smith" → ("Mary", "", "Smith")
```

### E2. Matching Strategy

**Multi-Tier Matching:**

**Tier 1: Exact Match (Highest Confidence)**
- Email exact match (case-insensitive)
- Phone exact match (E.164 format)
- Confidence: 100%

**Tier 2: Strong Match (High Confidence)**
- Name (first + last) + ZIP exact match
- Name (first + last) + Phone partial match (last 7 digits)
- Confidence: 90%

**Tier 3: Fuzzy Match (Medium Confidence)**
- Name Levenshtein distance ≤ 2 + Address similarity ≥ 80%
- Email domain match + Name similarity ≥ 85%
- Confidence: 70%

**Implementation:**
```python
from Levenshtein import distance

def calculate_match_score(person_a, person_b):
    """
    Returns match score 0-100
    """
    score = 0
    
    # Email match (40 points)
    if person_a['email'] and person_b['email']:
        if person_a['email'].lower() == person_b['email'].lower():
            score += 40
    
    # Phone match (30 points)
    if person_a['phone_e164'] and person_b['phone_e164']:
        if person_a['phone_e164'] == person_b['phone_e164']:
            score += 30
    
    # Name similarity (20 points)
    name_a = f"{person_a['first_name']} {person_a['last_name']}".lower()
    name_b = f"{person_b['first_name']} {person_b['last_name']}".lower()
    name_distance = distance(name_a, name_b)
    if name_distance <= 2:
        score += 20 - (name_distance * 5)
    
    # Address similarity (10 points)
    if person_a['zip'] and person_b['zip']:
        if person_a['zip'] == person_b['zip']:
            score += 10
    
    return score
```

### E3. Golden Record Resolution Rules

**Business Rules (Priority Order):**

1. **Most Recent Source Wins:** Data from newer import preferred
2. **Most Complete Record Wins:** Record with more filled fields preferred
3. **Verified Data Wins:** Email/phone marked as verified takes precedence
4. **Manual Override:** User-marked golden record always wins

**Resolution Algorithm:**
```python
def resolve_golden_record(duplicate_group):
    """
    Given a list of duplicate person records, determine the golden record
    """
    # Sort by business rules priority
    sorted_records = sorted(duplicate_group, key=lambda x: (
        x['is_manually_verified'],  # Manual verification highest priority
        x['last_updated_date'],     # Most recent second
        count_filled_fields(x),     # Most complete third
        x['email_verified'],        # Verified email fourth
        x['phone_verified']         # Verified phone fifth
    ), reverse=True)
    
    # First record is golden
    golden = sorted_records[0]
    
    # Mark golden record
    update_person(golden['id'], is_original=True)
    
    # Link all duplicates to golden
    for duplicate in sorted_records[1:]:
        create_duplicate_link(
            original_id=golden['id'],
            duplicate_id=duplicate['id'],
            match_score=calculate_match_score(golden, duplicate),
            match_reason='dedup_engine'
        )
        update_person(duplicate['id'], is_original=False)
    
    return golden['id']
```

### E4. Audit Trail & Lineage

**Every deduplication action is logged:**

```sql
CREATE TABLE dedup_audit_log (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    job_id VARCHAR(50),
    original_person_id BIGINT,
    duplicate_person_id BIGINT,
    match_score DECIMAL(5,2),
    match_reason VARCHAR(100),
    merged_fields JSON,  -- Which fields were merged
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_job_id (job_id),
    INDEX idx_original (original_person_id),
    INDEX idx_duplicate (duplicate_person_id)
);
```

**Data Lineage:**
```json
{
  "person_id": 12345,
  "lineage": [
    {
      "import_id": "imp_001",
      "source_file": "legacy_mysql_export.sql",
      "imported_at": "2026-02-01T10:00:00Z",
      "fields_imported": ["first_name", "last_name", "email"]
    },
    {
      "import_id": "imp_047",
      "source_file": "campaign_2026_q1.csv",
      "imported_at": "2026-02-15T14:30:00Z",
      "fields_updated": ["phone", "address"],
      "merge_reason": "email_match"
    }
  ],
  "merged_from": [67890, 54321],
  "current_status": "golden_record"
}
```

---


