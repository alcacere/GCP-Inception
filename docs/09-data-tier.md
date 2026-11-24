# 09 · Data Tier: Cloud SQL, Cloud Storage, Memorystore

In the original project, the stateful services set the hardest constraints: UIDs,
idempotent initialization, bind-mounted volumes. Managed data services remove most of
that work, but add new questions: backup policy, recovery objectives, private
connectivity and cost.

---

## Cloud SQL for MySQL

**What it is.** Managed MySQL, PostgreSQL or SQL Server. Google handles patching,
backups, replication, failover (with HA) and storage growth. **MariaDB is not
offered**, so WordPress moves to **MySQL 8**, which it fully supports
([ADR 0003](adr/0003-cloud-sql-mysql.md)).

**How this project will implement it**

| Setting | Dev value | Prod value (Phase 3) | Why |
|---|---|---|---|
| Edition | Enterprise | Enterprise | Enterprise Plus is for high-end workloads |
| Tier | `db-f1-micro` (shared core) | `db-custom-1-3840` | Shared core is the cheapest option and has no SLA, which is fine for dev |
| Availability | Zonal | Regional (HA) | HA doubles the cost. Not worth it for dev |
| Public IP | **Off** | **Off** | Private service access only ([06](06-networking.md)) |
| SSL mode | `ENCRYPTED_ONLY` | `ENCRYPTED_ONLY` | Encryption in transit even inside the VPC |
| Automated backups | Daily, 7 retained | Daily, 30 retained | |
| Point-in-time recovery | On | On | Uses binary logs to restore to any second in the retention window |
| Deletion protection | Off (so `make destroy` works) | On | |
| Maintenance window | Sunday 03:00 | Sunday 03:00 | Predictable restarts |

**Connecting from Cloud Run.** Two supported options:

1. **Private IP over Direct VPC egress** with TLS. This is the planned option.
2. **The Cloud SQL Auth Proxy / built-in connector** (`/cloudsql/<connection-name>`
   Unix socket), which handles TLS and IAM automatically.

**Recovery objectives (to verify in the runbook):** RPO ≈ seconds with PITR,
RTO ≈ the time to restore to a new instance, measured during the Phase 3 restore test.

**What replaces the MariaDB entrypoint?** The bootstrap SQL (create database, create
user, grant privileges) becomes Terraform resources (`google_sql_database`,
`google_sql_user`) plus the secret-seeding script for the password. The
least-privilege grant (`wordpress` user on the `wordpress` database only) is kept.

**References**
- Cloud SQL for MySQL: <https://cloud.google.com/sql/docs/mysql>
- Editions: <https://cloud.google.com/sql/docs/mysql/editions-intro>
- Backups: <https://cloud.google.com/sql/docs/mysql/backup-recovery/backups>
- Point-in-time recovery: <https://cloud.google.com/sql/docs/mysql/backup-recovery/pitr>
- High availability: <https://cloud.google.com/sql/docs/mysql/high-availability>
- IAM database authentication: <https://cloud.google.com/sql/docs/mysql/iam-authentication>
- Importing data: <https://cloud.google.com/sql/docs/mysql/import-export/import-export-sql>

---

## Cloud Storage

**What it is.** Object storage: buckets hold immutable objects, accessed through an
API (not a file system). Storage classes (Standard, Nearline, Coldline, Archive)
trade storage price against access price.

**Planned buckets**

| Bucket | Purpose | Settings |
|---|---|---|
| `<project>-tfstate` | Terraform state | Versioning, public access prevention, deployer-only IAM |
| `<project>-uploads` | WordPress media | Versioning, soft delete, public access prevention, `sa-wordpress` = `objectUser` |
| `<project>-portfolio` | Static site behind the CDN | Public read (intentional), `Cache-Control` metadata |
| `<project>-migration` | One-time dump transfer | Lifecycle rule: delete after 7 days |

This also covers what the **FTP server** used to do: getting files in and out of the
site. `gcloud storage cp` / `rsync` with IAM replaces a password-protected protocol
that sent credentials in cleartext ([ADR 0005](adr/0005-retire-ftp.md)).

**References**
- Cloud Storage overview: <https://cloud.google.com/storage/docs/introduction>
- Storage classes: <https://cloud.google.com/storage/docs/storage-classes>
- Object versioning: <https://cloud.google.com/storage/docs/object-versioning>
- Soft delete: <https://cloud.google.com/storage/docs/soft-delete>
- Lifecycle management: <https://cloud.google.com/storage/docs/lifecycle>

---

## Memorystore (optional)

**What it is.** Managed Redis-compatible in-memory stores (Memorystore for Redis, and
the newer Memorystore for Valkey). They are reached over private IP, with AUTH and
in-transit encryption.

**Why it is optional.** The smallest Memorystore instance costs much more per month
than the rest of the dev environment together (see [13](13-cost-management.md)).
WordPress works without an object cache. The plan:

- `var.enable_cache = false` by default. The `cache` module uses `count`.
- When it is turned on for a demo, WordPress reads the host and AUTH string from
  Secret Manager, and the `redis-cache` plugin is activated by the entrypoint, as in
  the original project.
- The measurable question to answer: *does the cache change p95 latency enough to be
  worth its cost at this traffic level?*

**References**
- Memorystore for Redis: <https://cloud.google.com/memorystore/docs/redis>
- Memorystore for Valkey: <https://cloud.google.com/memorystore/docs/valkey>

---

## Backup and restore summary

| Data | Backup mechanism | Restore test |
|---|---|---|
| Database | Cloud SQL automated backups + PITR | Restore to a scratch instance and compare row counts (scheduled in Phase 3) |
| Uploads | Object versioning + soft delete | Delete an object, then restore it |
| Phase 1 data disk | Snapshot schedule | Create a disk from a snapshot, attach it, verify |
| Infrastructure | Terraform code in git + versioned state | `make destroy && make apply` |
| Secrets | Secret Manager versions | Disable the current version, re-enable the previous one |

A backup that has never been restored is not a backup. Every row in this table gets a
tested procedure in the [runbook](12-operations-runbook.md).
