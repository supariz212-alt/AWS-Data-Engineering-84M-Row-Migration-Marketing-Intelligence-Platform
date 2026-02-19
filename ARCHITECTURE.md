# AWS Data Engineering Architecture – 84M Record Consolidation

## 1. Infrastructure Layer

- Private AWS VPC
- Aurora MySQL (Multi-AZ)
- 2 Read Replicas
- ElastiCache Redis
- S3 (raw + processed storage)
- AWS Glue
- IAM role-based access control

---

## 2. Migration Strategy

### Phase A – Initial Bulk Migration
- Snapshot export from legacy MySQL
- CSV normalization from SQLite
- Chunked batch ingestion (100k rows per batch)
- Atomic transaction control

### Data Validation
- Row count verification per source
- Schema validation checks
- Referential integrity enforcement
- Reconciliation reporting

---

## 3. Ingestion Engine Design

### Atomic Upsert Logic


- Idempotent loads
- Match keys: email, phone_e164
- Conflict resolution based on latest timestamp

### Version Tracking

Tables:
- source_files
- import_batches
- record_versions

Each batch includes:
- Row count
- Error log
- Import timestamp
- Processing duration

---

## 4. Deduplication Engine

Golden Record logic:

- Exact match: email
- Normalized phone match
- Fuzzy name match (Levenshtein threshold)
- Business rule scoring model

Flags:
- is_original
- is_duplicate
- merged_into_id

Full audit maintained in:
- dedupe_log
- change_history

---

## 5. Compliance Engine

Automated checks:
- DNC registry scrub
- Opt-out list filtering
- Frequency cap enforcement
- Status tagging

---

## 6. Dashboard Performance Design

Target: <1.5s query response

Strategies:
- Materialized views for priority lists
- Indexed filtering columns
- Redis cache for repeated queries
- Pagination with limit-offset optimization
- Async background pre-aggregation

---

## 7. Monitoring & Observability

- CloudWatch metrics
- Slow query logging
- Glue job failure alerts
- Import anomaly detection
- Dashboard performance monitoring

---

## 8. Disaster Recovery

- Daily automated snapshots
- 7-day retention policy
- Backup validation tests
- Documented recovery procedure

---

This architecture ensures scalability, compliance, auditability, and high-performance operations at 84M+ scale.
