# Inception on GCP: From One VM to a Managed Cloud Platform

## What this project is

The original [Inception](https://github.com/alcacere/inception) builds a self-hosted
web platform entirely from source: Packer builds a Debian VM, Vagrant runs it, and
Docker Compose runs nine hand-written containers (NGINX, WordPress/PHP-FPM, MariaDB,
Redis, vsftpd, Adminer, a static site, Prometheus and Blackbox Exporter).

This project takes **the same application** and runs it on Google Cloud Platform. It
replaces the parts I had to build by hand with the managed service a cloud team would
actually use, and it documents every replacement as a deliberate decision:

```
Phase 0  Foundations     project, budget, IAM, Terraform state, keyless CI
Phase 1  Rehost          the original compose stack on a Compute Engine VM, built as code
Phase 2  Replatform      Cloud Run + Cloud SQL + Load Balancer + Cloud Armor + Monitoring
Phase 3  Deliver         CI/CD pipeline, scanning, approvals, second environment
Phase 4  Stretch         GKE Autopilot, Binary Authorization, certification
```

## Why it matters professionally

A junior cloud engineer is expected to provision, secure, operate and pay for cloud
infrastructure, not just to run containers. Each phase maps to that work:

| Skill area | What this project does | What a cloud team calls it |
|---|---|---|
| **Cloud foundations** | Project, billing, budgets, APIs, regions, labels | Landing zone, resource hierarchy |
| **Infrastructure as Code** | Terraform modules, remote state, environments; Packer images for GCE | IaC, golden images, GitOps |
| **Identity and access** | One service account per workload, least-privilege roles, Workload Identity Federation, IAP | IAM, zero trust, keyless auth |
| **Networking** | Custom VPC, firewall, Cloud NAT, private service access, Cloud DNS | Cloud network engineering |
| **Edge and security** | Global Application Load Balancer, managed TLS, SSL policies, Cloud Armor (WAF) | Ingress, WAF, DDoS protection |
| **Compute** | Compute Engine (IaaS) and Cloud Run (serverless containers), compared directly | Choosing a compute platform |
| **Managed data** | Cloud SQL with backups and PITR, Cloud Storage, optional Memorystore | DBaaS, durability, RPO/RTO |
| **Delivery** | GitHub Actions, Artifact Registry, deploy by digest, rollback | CI/CD, supply-chain security |
| **Observability** | Cloud Logging, Monitoring, uptime checks, alerting, SLOs, Grafana dashboards as code | SRE practices |
| **Cost** | Budgets, cost per phase, toggles, teardown | FinOps |

## What carries over from the original project

These principles don't change. Only their implementation does:

| Inception principle | How it survives on GCP |
|---|---|
| Built, not pulled | Images are still built from my own Dockerfiles and stored in Artifact Registry |
| One process per container | Cloud Run sidecars: NGINX and PHP-FPM run as separate containers in one service |
| A single encrypted entry point | One load balancer, TLS 1.2+ only, no port 80, backends not publicly reachable |
| Secrets never in code or images | Secret Manager, mounted as files, so the application still reads `/run/secrets/...`-style paths |
| State survives anything but an explicit wipe | Cloud SQL backups, PITR, versioned buckets, deletion protection |
| Observable by default | Uptime checks probe from several regions (the job Blackbox did), and Grafana replaces the Prometheus UI as the place to look at everything |
| Reproducible from nothing | `make bootstrap && make apply` in an empty project |

## Documentation map

| # | Document | Topic |
|---|---|---|
| 01 | [Architecture and design](01-architecture.md) | Target architecture, request flow, repository layout |
| 02 | [Migration strategy](02-migration-strategy.md) | Rehost vs replatform vs refactor, component mapping |
| 03 | [GCP foundations](03-gcp-foundations.md) | Resource hierarchy, projects, billing, APIs, regions, `gcloud` |
| 04 | [Infrastructure as Code](04-infrastructure-as-code.md) | Terraform, remote state, modules, environments, Packer on GCE |
| 05 | [Identity and security](05-identity-and-security.md) | IAM, service accounts, WIF, Secret Manager, IAP |
| 06 | [Networking](06-networking.md) | VPC, firewall, Cloud NAT, private service access, Cloud DNS |
| 07 | [Edge: load balancing, TLS, WAF](07-edge-load-balancing-tls.md) | Application LB, managed certificates, SSL policies, Cloud Armor, CDN |
| 08 | [Compute and containers](08-compute-and-containers.md) | Artifact Registry, Compute Engine, Cloud Run, GKE |
| 09 | [Data tier](09-data-tier.md) | Cloud SQL, Cloud Storage, Memorystore, migration, backups |
| 10 | [CI/CD](10-ci-cd.md) | GitHub Actions, WIF, scanning, deploy by digest, rollback |
| 11 | [Observability](11-observability.md) | Logging, Monitoring, uptime checks, alerting, SLOs, Grafana |
| 12 | [Operations runbook](12-operations-runbook.md) | Deploy, verify, day-2 operations, teardown, troubleshooting |
| 13 | [Cost management](13-cost-management.md) | Budgets, cost per phase, free tier, toggles |
| 14 | [Best practices and roadmap](14-best-practices-and-roadmap.md) | Architecture Framework, trade-offs, certification map |
| — | [Architecture Decision Records](adr/README.md) | Why each major choice was made |
| — | [Sources](sources.md) | All references |

Each technology page follows the outline used in the original project: **what it
is**, **why it is used here**, **how it works**, **how this project will implement
it**, and **references** to official documentation. Pages that describe something not
built yet say *will*, and switch to the present tense once it is implemented.
