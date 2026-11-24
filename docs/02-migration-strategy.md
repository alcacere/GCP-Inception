# 02 · Migration Strategy

## The vocabulary: the "Rs" of migration

Cloud migrations are usually described with a small set of strategies (Google's
migration guides and AWS's "7 Rs" use nearly the same list):

| Strategy | Meaning | Effort | Benefit |
|---|---|---|---|
| **Rehost** ("lift and shift") | Move the workload as it is onto cloud VMs | Low | Fast. You learn the cloud basics without changing the application |
| **Replatform** ("lift and optimize") | Swap pieces for managed services, keep the application | Medium | Less operational work: patching, backups and scaling are handled for you |
| **Refactor** ("re-architect") | Change the application to be cloud-native | High | Best scalability and cost, but it is a rewrite |
| **Retire** | Remove what isn't needed any more | — | Less to run and secure |
| **Retain** | Keep it where it is | — | — |

**This project does rehost → replatform**, plus a few targeted **retire** decisions.
It doesn't refactor WordPress itself: the application is the constant, and the
infrastructure is what changes. That is the realistic scenario for a junior cloud
engineer, who usually migrates applications they didn't write.

Doing *both* rehost and replatform, instead of jumping straight to Cloud Run, is
deliberate ([ADR 0002](adr/0002-phased-migration.md)):

1. Phase 1 gives a working cloud baseline quickly, which reduces risk.
2. It practices the IaaS skills (VMs, images, disks, firewall, IAP) that the
   certification and most job interviews cover.
3. It produces a **measurable comparison** of cost, deploy time and operational
   effort between the two phases.

## Component mapping

| Original component | Phase 1 (rehost) | Phase 2 (replatform) | Decision |
|---|---|---|---|
| Packer + preseed (Debian ISO) | Packer `googlecompute` builder from a public Debian image | Not needed (no VMs) | Google provides base images, so the ISO and preseed step disappears |
| Vagrant + VirtualBox | Terraform `google_compute_instance` | Not needed | [ADR 0001](adr/0001-terraform-for-iac.md) |
| Docker Compose | Compose on the VM | Cloud Run (service + sidecars) | Same images, a different scheduler |
| NGINX (TLS + routing) | Unchanged container + Let's Encrypt | Load balancer (TLS, routing) + NGINX sidecar (FastCGI) | [ADR 0004](adr/0004-nginx-as-cloud-run-sidecar.md) |
| Private two-tier CA | Let's Encrypt | Google-managed certificate | A public site needs publicly trusted certificates |
| WordPress + PHP-FPM | Unchanged | Immutable image, uploads on Cloud Storage | Cloud Run file systems are ephemeral |
| MariaDB | Unchanged container, data on a persistent disk | Cloud SQL for MySQL | [ADR 0003](adr/0003-cloud-sql-mysql.md) |
| Redis | Unchanged container | Memorystore, optional | Cost-gated. WordPress works without it |
| Docker secrets (`secrets/*.txt`) | Secret Manager → tmpfs files at boot | Secret Manager → Cloud Run secret volumes | The file-based contract is kept |
| vsftpd | Retired | Retired | [ADR 0005](adr/0005-retire-ftp.md) |
| Adminer | Kept, reachable only through an IAP tunnel | Retired in favour of Cloud SQL Studio (optionally Adminer behind IAP) | An admin UI must never be public |
| Static site (darkhttpd) | Unchanged container | Backend bucket + Cloud CDN | No compute is needed for static files |
| Prometheus + Blackbox | Replaced by a Grafana container reading Cloud Monitoring (Ops Agent + uptime checks), reached through IAP | Grafana on Cloud Run behind IAP + Cloud Monitoring uptime checks, alerts, SLOs | [ADR 0007](adr/0007-cloud-monitoring-over-self-hosted-prometheus.md), [ADR 0008](adr/0008-grafana-replaces-blackbox.md) |
| `make` | `make` → Terraform/Packer | `make` + GitHub Actions | [ADR 0006](adr/0006-keyless-ci-with-wif.md) |

## Risks and how the plan handles them

| Risk | Mitigation |
|---|---|
| Unexpected bill | A budget is set before anything else. Expensive resources are toggles. `make destroy` exists from day one. See [13](13-cost-management.md) |
| Data loss during the MariaDB → Cloud SQL move | Dump and restore with checksum verification, while the Phase 1 stack keeps running until Phase 2 is verified (see [09](09-data-tier.md)) |
| WordPress writes to its own disk (plugins, uploads) | Immutable image for code. Only `uploads/` is writable, and it goes to Cloud Storage |
| Cold starts on Cloud Run | `min-instances=0` in dev, documented trade-off. Startup CPU boost |
| Locking myself out (IAM mistakes) | Bootstrap keeps my user as project Owner. Deployer permissions are added only through Terraform |
| Secrets leaking into Terraform state | Terraform creates the *secret containers* only. Values are added out-of-band (see [05](05-identity-and-security.md)) |

## Migration steps for the data (Phase 1 → Phase 2)

```
1. Freeze writes    WordPress maintenance mode on the Phase 1 stack
2. Export           mariadb-dump --single-transaction --routines --triggers wordpress
3. Upload           gcloud storage cp dump.sql.gz gs://<project>-migration/
4. Import           gcloud sql import sql <instance> gs://…/dump.sql.gz --database=wordpress
5. Media            gcloud storage rsync -r wp-content/uploads gs://<project>-uploads/
6. Verify           row counts per table + a checksum of the uploads, compared on both sides
7. Cut over         Point DNS at the load balancer (low TTL set 24 h before)
8. Keep Phase 1     for a rollback window, then terraform destroy -target=module.vm
```

## References

- Migrate to Google Cloud, getting started: <https://cloud.google.com/architecture/migration-to-gcp-getting-started>
- Cloud Architecture Center: <https://cloud.google.com/architecture>
- Google Cloud Well-Architected Framework: <https://cloud.google.com/architecture/framework>
