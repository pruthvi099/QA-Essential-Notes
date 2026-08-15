# 05 — SQL & Database Testing

Backend and data validation for QA — from core SQL fluency to transaction integrity and migration testing. Read [00-start-here](../00-start-here/) first for general testing fundamentals; this folder assumes comfort with basic testing concepts, not SQL.

## Notes

1. [SQL Fundamentals for QA](./sql-fundamentals-for-qa.md) — `SELECT`, `WHERE`, and the query patterns used constantly for validation
2. [Joins for Data Validation](./joins-for-data-validation.md) — Cross-table checks and finding orphaned records
3. [Aggregations & Data Integrity Checks](./aggregations-and-data-integrity-checks.md) — `GROUP BY`/`HAVING` for duplicates and summary-level bugs
4. [Backend Verification Testing](./backend-verification-testing.md) — Translating business rules into SQL verification queries
5. [API-to-Database Validation](./api-to-database-validation.md) — Cross-referencing API responses against real DB state
6. [Transaction & Consistency Testing](./transaction-and-consistency-testing.md) — ACID properties, rollback, and race-condition testing
7. [Test Data Seeding with SQL](./test-data-seeding-with-sql.md) — Bulk seed/reset scripts for repeatable environments
8. [Database Migration Testing](./database-migration-testing.md) — Verifying schema changes don't break data or compatibility

## Related

- [01 — Manual Testing](../01-manual-testing/) — general test data management principles this folder extends
- [04 — API Testing](../04-api-testing/) — API-level testing this folder's DB checks complement