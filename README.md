# Inception on Google Cloud

*A port of my [Inception](https://github.com/alcacere/inception) project (a containerized
WordPress platform built from a bare Debian ISO for the 42 curriculum) to Google
Cloud Platform. It is built as a cloud engineering portfolio project.*

> **Status: planning.** This repository contains only the plan and the learning
> documentation for now. Implementation follows the phases in [PLAN.md](PLAN.md).

## The idea in one diagram

```
Original Inception (local)                 Inception on GCP (target)
──────────────────────────                 ─────────────────────────
Packer + Vagrant + VirtualBox      ──►     Terraform (+ Packer for the Phase 1 image)
Docker Compose on one VM           ──►     Compute Engine (Phase 1) → Cloud Run (Phase 2)
NGINX + private CA                 ──►     External Application LB + Google-managed TLS
                                           + Cloud Armor (NGINX kept as a sidecar)
MariaDB container                  ──►     Cloud SQL for MySQL (private IP only)
Redis container                    ──►     Memorystore (optional, cost-gated)
Docker secrets (files)             ──►     Secret Manager (still mounted as files)
vsftpd                             ──►     Cloud Storage (removed as a service, see ADR 0005)
Static site (darkhttpd)            ──►     Cloud Storage bucket + Cloud CDN
Prometheus + Blackbox              ──►     Grafana (Cloud Run, behind IAP) on top of Cloud
                                           Monitoring: uptime checks, alerts, SLOs
make                               ──►     make + GitHub Actions (keyless via WIF)
```

The original project proved I could build each layer by hand. This one shows I can
run the same system on a public cloud, so the focus shifts to **managed services,
identity, networking, automation, observability and cost**.

## Where to start

| Read this | For |
|---|---|
| [PLAN.md](PLAN.md) | Phases, milestones, definitions of done, open decisions |
| [docs/00-overview.md](docs/00-overview.md) | What the project is and the documentation map |
| [docs/01-architecture.md](docs/01-architecture.md) | Target architecture, request flow, repository layout |
| [docs/02-migration-strategy.md](docs/02-migration-strategy.md) | How each Inception component maps to GCP, and why |
| [docs/adr/](docs/adr/README.md) | Architecture Decision Records |
| [docs/sources.md](docs/sources.md) | All references in one place |

## Planned interface

The Makefile stays the single entry point, as in the original project:

```sh
make bootstrap   # one-time: project APIs, Terraform state bucket, CI identity
make plan        # terraform plan for the selected environment
make apply       # terraform apply
make build       # build images locally
make push        # push images to Artifact Registry
make deploy      # deploy the images by digest
make smoke       # run the end-to-end verification checks
make destroy     # tear everything down (the cost safety net)
```
