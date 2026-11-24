# 14 · Best Practices, Trade-offs and Roadmap

## Mapping to the Google Cloud Well-Architected Framework

Google's Well-Architected Framework groups best practices into pillars. Each one
maps to concrete decisions in this project. This table is useful for a README, an
interview, or a design review:

| Pillar | What this project does |
|---|---|
| **Operational excellence** | Everything in Terraform, one Makefile interface, CI/CD with plan-on-PR, a runbook with tested procedures, structured logs, Grafana dashboards versioned in git |
| **Security, privacy and compliance** | Least-privilege SA per workload, no SA keys (WIF), Secret Manager, private-IP-only database, IAP instead of public SSH/admin, Cloud Armor WAF, TLS 1.2+, audit logs |
| **Reliability** | Regional managed services, health-gated rollouts, revision-based rollback, PITR backups with restore tests, uptime checks and SLOs |
| **Cost optimization** | Budgets first, labels, scale-to-zero, cost-gated optional resources, measured cost comparison between phases |
| **Performance optimization** | CDN for static content, object cache as a measured experiment, right-sizing from metrics instead of guesses |

**References**
- Google Cloud Well-Architected Framework: <https://cloud.google.com/architecture/framework>

---

## Design trade-offs

| Decision | Alternative | Why this choice |
|---|---|---|
| Rehost first, then replatform | Go straight to Cloud Run | Lower risk, practices IaaS, produces a measured comparison ([ADR 0002](adr/0002-phased-migration.md)) |
| Cloud Run | GKE Autopilot | No cluster to operate for one app, scale to zero. GKE is a stretch goal ([08](08-compute-and-containers.md)) |
| Cloud SQL for MySQL | MariaDB on a VM | Managed backups, PITR, patching. MariaDB isn't offered ([ADR 0003](adr/0003-cloud-sql-mysql.md)) |
| NGINX sidecar | Apache-based PHP image | Keeps one process per container and reuses the original config ([ADR 0004](adr/0004-nginx-as-cloud-run-sidecar.md)) |
| Retire FTP | SFTP server on a VM | Cleartext credentials, an always-on host. Cloud Storage + IAM is safer ([ADR 0005](adr/0005-retire-ftp.md)) |
| GitHub Actions + WIF | Cloud Build / SA keys | Common in industry, keyless ([ADR 0006](adr/0006-keyless-ci-with-wif.md)) |
| Cloud Monitoring | Self-hosted Prometheus | Nothing to run. Probes from outside the network ([ADR 0007](adr/0007-cloud-monitoring-over-self-hosted-prometheus.md)) |
| Grafana (self-built, behind IAP) | Cloud Monitoring console only, Grafana Cloud | A widely used tool, dashboards as code in git, no passwords thanks to IAP ([ADR 0008](adr/0008-grafana-replaces-blackbox.md)) |
| Private service access | Private Service Connect | Most common pattern first. PSC is on the roadmap |
| Memorystore off by default | Always on | It would dominate the bill. Its value is measured, not assumed |

---

## Roadmap after Phase 3

- **GKE Autopilot + Helm** for the same images, to compare with Cloud Run.
- **Binary Authorization**: only CI-attested images can be deployed.
- **Cloud Deploy** for `dev` → `prod` promotion with approvals.
- **Private Service Connect** for Cloud SQL, removing VPC peering.
- **Organization policies** (no SA keys, no external IPs, allowed regions) if an
  organization is created (Cloud Identity Free).
- **Multi-region** design on paper: what would change for 99.9 %, and at what cost.
- **Terraform tests** (`terraform test`) for the modules.

---

## Certification map: Associate Cloud Engineer

The Associate Cloud Engineer certification is the usual entry-level target for this
kind of role. Its exam guide is organized in sections that this project covers in
practice. Check the current guide before studying, because it is revised periodically.

| Exam guide section | Where this project practices it |
|---|---|
| Setting up a cloud solution environment | Phase 0: projects, billing, budgets, APIs, gcloud, IAM basics ([03](03-gcp-foundations.md)) |
| Planning and implementing a cloud solution | Compute choices, Cloud SQL, Cloud Storage, VPC, load balancing, Terraform ([04](04-infrastructure-as-code.md)–[09](09-data-tier.md)) |
| Ensuring successful operation of a cloud solution | Snapshots, backups, Cloud Run revisions, logging, monitoring, alerts ([11](11-observability.md), [12](12-operations-runbook.md)) |
| Configuring access and security | IAM roles, service accounts, impersonation, audit logs ([05](05-identity-and-security.md)) |

**References**
- Associate Cloud Engineer: <https://cloud.google.com/learn/certification/cloud-engineer>
- Google Cloud Skills Boost: <https://www.cloudskillsboost.google/>

---

## How to present this project

A short version for a CV or LinkedIn, to update with real numbers at the end:

> *Migrated a containerized WordPress platform from a local VM to Google Cloud in two
> phases (Compute Engine, then Cloud Run + Cloud SQL), fully in Terraform with keyless
> GitHub Actions CI/CD (Workload Identity Federation). Private-IP-only database,
> Secret Manager, global HTTPS load balancer with Cloud Armor, uptime checks and SLO
> burn-rate alerting, with Grafana dashboards as code behind Identity-Aware Proxy. Documented every decision in ADRs and measured cost and
> operational trade-offs between IaaS and serverless.*

Interview stories this project should be able to tell, with evidence from the repo:
1. A security decision: no SA keys, and the WIF attribute condition.
2. A cost decision: Memorystore off, and the LB fixed cost.
3. A failure: a broken deploy, how the alert fired, how the rollback worked.
4. A migration: moving data from MariaDB to Cloud SQL, and how it was verified.
5. A "why not": why not Kubernetes for this workload.

## Lessons learned

*To be written during implementation, as in the original project.*
