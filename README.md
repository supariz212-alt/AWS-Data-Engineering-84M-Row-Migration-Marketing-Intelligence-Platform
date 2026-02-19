# 84M Record Migration – AWS Data Engineering Platform

Production-grade AWS data architecture for consolidating 84+ million records into a unified Source of Truth with automated ingestion, deduplication, compliance, and sub-1.5s query performance.

---

## What This Repository Demonstrates

This repository outlines a reference architecture and execution framework for large-scale data consolidation projects involving:

- 68M+ legacy database records (MySQL / SQLite)
- 15M+ historical CSV campaign records
- Continuous ingestion requirements
- Golden Record deduplication logic
- Compliance filtering (DNC / Opt-Out)
- High-performance operational dashboard

---

## Core Capabilities

✓ Zero data loss migration strategy  
✓ Atomic upsert ingestion engine (idempotent loads)  
✓ Versioned import tracking & reconciliation  
✓ Fuzzy matching Golden Record algorithm  
✓ Full audit trail and change history  
✓ Sub-1.5s query performance architecture  
✓ Private VPC AWS infrastructure design  

---

## Technical Stack

- AWS Aurora MySQL (RDS)
- AWS Glue (ETL orchestration)
- Python / PySpark
- ElastiCache Redis (query acceleration)
- FastAPI (backend services)
- React / Next.js (Operations Dashboard)

---

## Repository Structure

/sql/ - Database schema, indexes, partitions
/ingest/ - Atomic ingestion engine
/dedupe/ - Golden record logic
/api/ - Dashboard & integration endpoints
/docs/ - Architecture diagrams & runbooks

---

## Performance Strategy

- Table partitioning by HASH(person_id)
- Composite indexes: (email), (phone_e164), (last_name, zip)
- Aurora read replicas for dashboard traffic
- Redis caching for filtered list queries
- Query profiling via Performance Insights

---

## Migration Integrity Controls

- Pre/post row count validation
- SHA256 checksum comparison
- Sampling-based verification
- Batch-level validation reports
- Automated reconciliation logs

---

This architecture is designed for enterprise-scale data unification projects requiring performance, compliance, and auditability.
