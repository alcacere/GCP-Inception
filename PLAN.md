# Project Plan

This plan moves Inception to GCP in phases. Each phase ends in a working, demonstrable,
**torn-down-able** system. A phase is done only when its *definition of done* is met
and the docs are updated.

The migration follows the standard sequence of **rehost → replatform → (optionally)
refactor**. The reasoning is in [docs/02-migration-strategy.md](docs/02-migration-strategy.md).

---

## Phase 0 · Foundations

**Goal:** a safe, cost-controlled GCP environment that Terraform and CI can work in.

- [ ] Create a GCP project (`inception-gcp-<suffix>`) and link billing
- [ ] **Budget with alerts** at 50 / 90 / 100 % *before anything else is deployed*
- [ ] Install and configure `gcloud`, Terraform (or OpenTofu), `tflint`
- [ ] `infra/bootstrap/`: enable APIs, create the Terraform state bucket (versioned,
      uniform bucket-level access, public access prevention)
- [ ] Workload Identity Federation pool and provider for GitHub Actions (no service account keys)
- [ ] Terraform deployer service account with least-privilege roles
- [ ] Artifact Registry Docker repository
- [ ] Labels/tags convention (`project=inception`, `env=dev`, `phase=N`)

**Definition of done**
- `make bootstrap` is idempotent: running it twice changes nothing
- A GitHub Actions job authenticates to GCP with WIF and runs `terraform plan`
- No JSON key exists for any service account (`gcloud iam service-accounts keys list`)
- A test budget alert email has been received

Docs: [03](docs/03-gcp-foundations.md), [04](docs/04-infrastructure-as-code.md), [05](docs/05-identity-and-security.md), [13](docs/13-cost-management.md)

---

## Phase 1 · Rehost ("lift and shift" to Compute Engine)

**Goal:** the *original* compose stack running on a GCE VM, built and managed as code.
This shows IaaS skills and creates a baseline to compare Phase 2 against.

- [ ] Packer `googlecompute` builder that reuses the original provisioning scripts
      (Docker install, users, SSH hardening) and produces a custom image family
- [ ] Terraform `network` module: custom VPC, subnet, firewall (443 only + IAP SSH), Cloud NAT
- [ ] Terraform `vm` module: VM from the image family, no external SSH, static external IP,
      dedicated service account, Shielded VM, OS Login
- [ ] Startup script that pulls secrets from **Secret Manager** into a tmpfs and writes
      them as files, so the compose `secrets:` block keeps working unchanged
- [ ] Persistent disk for `${DATA_PATH}` (data survives VM recreation), with a snapshot schedule
- [ ] Cloud DNS zone and A record. Let's Encrypt replaces the private CA (requires a domain)
- [ ] Ops Agent for logs and host metrics
- [ ] Blackbox and Prometheus containers replaced by a **Grafana** container that reads
      Cloud Monitoring. No public port: reached through an IAP tunnel, like SSH

**Definition of done**
- `make apply && make smoke` from a fresh clone gives a working HTTPS site
- `terraform destroy` then `apply` restores the site with data intact (from the disk or snapshot)
- SSH only works through IAP (`gcloud compute ssh --tunnel-through-iap`)
- TLS 1.1 is refused, port 80 is closed, and the DB port is unreachable from the internet

Docs: [06](docs/06-networking.md), [08](docs/08-compute-and-containers.md), [12](docs/12-operations-runbook.md)

---

## Phase 2 · Replatform (managed and serverless services)

**Goal:** replace self-managed pieces with managed services while keeping the
Inception principles: one process per container, a single TLS entry point, secrets as
files, and a private data tier.

- [ ] Images pushed to Artifact Registry, tagged with the git SHA and deployed **by digest**
- [ ] **Cloud SQL for MySQL** with private IP only, automated backups and PITR; data migrated from MariaDB
- [ ] **Cloud Run** `wordpress` service: NGINX ingress container + PHP-FPM sidecar,
      Direct VPC egress, secrets mounted as files from Secret Manager
- [ ] Immutable WordPress image (core + plugins baked in). Uploads go to Cloud Storage
- [ ] Static portfolio in a Cloud Storage **backend bucket** with Cloud CDN
- [ ] **Global external Application Load Balancer**: serverless NEG + backend bucket,
      Google-managed certificate, SSL policy (`MODERN`, minimum TLS 1.2), no port 80
- [ ] **Cloud Armor** security policy (OWASP preconfigured rules, rate limiting on `/wp-login.php`)
- [ ] Cloud Run ingress = `internal-and-cloud-load-balancing` (the `*.run.app` URL is closed)
- [ ] Database admin: Cloud SQL Studio (IAM-gated). Adminer behind IAP is an optional exercise
- [ ] Memorystore as an **optional** Terraform toggle, off by default (see [13](docs/13-cost-management.md))
- [ ] Cloud Monitoring: uptime checks (HTTPS + certificate expiry), alert policies,
      and an availability SLO with an error-budget burn-rate alert
- [ ] **Grafana** on Cloud Run at `/grafana/`, behind IAP, with data sources and dashboards
      provisioned from git (Cloud Monitoring + Cloud Logging data sources, read-only service account)

**Definition of done**
- No VM is involved in serving traffic
- Grafana returns a Google sign-in redirect to anyone without IAP access, and its dashboards
  come back intact after the service is redeployed (they live in git, not in the container)
- Cloud SQL has no public IP, and no secret value appears in any `describe` output or Terraform state
- `make smoke` passes against the load balancer
- A deliberate failure (bad image) triggers an alert, and a rollback to the previous revision restores service

Docs: [07](docs/07-edge-load-balancing-tls.md), [08](docs/08-compute-and-containers.md), [09](docs/09-data-tier.md), [11](docs/11-observability.md)

---

## Phase 3 · Delivery and hardening

**Goal:** turn the deployment into a pipeline anyone could operate.

- [ ] CI on pull requests: ShellCheck, hadolint, `terraform fmt/validate`, tflint,
      IaC scanning (Trivy config or Checkov), image build, Trivy image scan, `terraform plan` as a PR comment
- [ ] CD on `main`: build → push → deploy by digest → smoke test → automatic rollback on failure
- [ ] `terraform apply` gated behind a GitHub Environment with manual approval
- [ ] A second environment (`prod`) from the same modules, with different variables
- [ ] Scheduled backup-restore test (restore the Cloud SQL backup to a scratch instance)

Docs: [10](docs/10-ci-cd.md), [14](docs/14-best-practices-and-roadmap.md)

---

## Phase 4 · Stretch goals (pick by interest)

- [ ] GKE Autopilot + Helm chart (the Kubernetes version of the same stack)
- [ ] Binary Authorization (deploy only images that CI signed)
- [ ] Cloud Deploy for promotion across `dev` → `prod`
- [ ] Organization policy constraints (requires an organization)
- [ ] Managed Service for Prometheus, to reuse the original PromQL alert rules (queried from Grafana)
- [ ] Google Cloud Associate Cloud Engineer certification (see [14](docs/14-best-practices-and-roadmap.md))

---

## Open decisions

| # | Question | Default until decided |
|---|---|---|
| 1 | Domain name: buy a cheap domain, or use a subdomain I already own? | Needed from Phase 1 (managed certificates and Let's Encrypt both need one) |
| 2 | Region | `europe-west1` (lower-cost tier, every service used here is available). `europe-southwest1` (Madrid) if latency matters more than cost |
| 3 | Terraform or OpenTofu | Terraform (most common in job postings). Code stays compatible with both |
| 4 | CI system | GitHub Actions + WIF. Cloud Build is documented as the alternative |
| 5 | Keep Memorystore? | Off by default; turned on only for a demo |
| 6 | Free trial credits | Start Phase 1 inside the free trial window to absorb mistakes |
