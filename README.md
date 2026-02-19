# 84M Record Migration - AWS Data Engineering Solution

Production-grade data pipeline for consolidating 84 million records with automated deduplication and sub-1.5s query performance.

## What This Demonstrates

- **Ingestion Engine:** Atomic upsert logic with validation checklists
- **Deduplication:** Golden record algorithm with fuzzy matching
- **Performance:** Query optimization strategies for <1.5s response
- **Architecture:** Complete AWS infrastructure design

## Technical Stack

- AWS Aurora MySQL (RDS)
- AWS Glue (ETL orchestration)
- Python/PySpark (data processing)
- ElastiCache Redis (query acceleration)
- React/Next.js (Operations Dashboard)

## Repository Structure
```
/sql/           - Database schema and indexes
/ingest/        - Ingestion engine Python scripts
/dedupe/        - Deduplication algorithm
/api/           - FastAPI backend endpoints
/docs/          - Architecture diagrams and runbooks
```

## Key Features

✓ Zero data loss migration  
✓ Atomic upsert with conflict resolution  
✓ Fuzzy matching deduplication  
✓ DNC compliance scrubbing  
✓ Sub-1.5s query performance  
✓ Complete audit trail  

Built for Community Minerals - 84M Record Migration Project
