# 0003 · Replace MariaDB with Cloud SQL for MySQL

**Status:** Accepted

## Context
The original stack runs MariaDB in a container I built and initialize myself. Cloud
SQL offers MySQL, PostgreSQL and SQL Server, but not MariaDB. WordPress officially
supports MySQL 8.

## Decision
Use Cloud SQL for MySQL 8 (Enterprise edition, shared-core tier in dev) with private
IP only, enforced encryption in transit, automated backups and point-in-time recovery.
Data is migrated with `mariadb-dump` → Cloud Storage → `gcloud sql import`.

## Alternatives considered
- **MariaDB on a Compute Engine VM.** Full compatibility, but I would own patching,
  backups and failover again, which defeats the replatform phase.
- **MariaDB container on Cloud Run.** Not viable: Cloud Run is stateless.
- **Cloud SQL for PostgreSQL.** WordPress doesn't support it without plugins.

## Consequences
- A dump from MariaDB has to import cleanly into MySQL 8. Known differences
  (collations, `utf8mb4` defaults, MariaDB-only syntax) are checked during the
  migration rehearsal.
- No scale to zero: Cloud SQL costs money every hour it exists. It can be stopped
  when idle ([13](../13-cost-management.md)).
- Database initialization moves from an entrypoint script to Terraform resources.
