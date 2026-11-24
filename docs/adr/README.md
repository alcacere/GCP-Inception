# Architecture Decision Records

An ADR records **one** significant decision: the context, the choice, the
alternatives that were rejected, and the consequences. ADRs are not edited once
accepted. A changed decision gets a new ADR that *supersedes* the old one, so the
history of *why* stays readable.

Format: [Michael Nygard's template](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions)
(Status, Context, Decision, Consequences), plus an "Alternatives considered" section.

| # | Decision | Status |
|---|---|---|
| [0001](0001-terraform-for-iac.md) | Use Terraform for all cloud infrastructure | Accepted |
| [0002](0002-phased-migration.md) | Migrate in phases: rehost on Compute Engine, then replatform | Accepted |
| [0003](0003-cloud-sql-mysql.md) | Replace MariaDB with Cloud SQL for MySQL | Accepted |
| [0004](0004-nginx-as-cloud-run-sidecar.md) | Keep NGINX as the Cloud Run ingress container with a PHP-FPM sidecar | Proposed |
| [0005](0005-retire-ftp.md) | Retire the FTP service | Accepted |
| [0006](0006-keyless-ci-with-wif.md) | Authenticate CI with Workload Identity Federation, never SA keys | Accepted |
| [0007](0007-cloud-monitoring-over-self-hosted-prometheus.md) | Use Cloud Monitoring instead of self-hosted Prometheus | Accepted |
| [0008](0008-grafana-replaces-blackbox.md) | Grafana replaces Blackbox Exporter and the Prometheus UI | Accepted |

"Proposed" means the decision depends on something that will be validated during
implementation (here: sidecar startup behaviour with PHP-FPM).

**References**
- ADR GitHub organization (templates and tooling): <https://adr.github.io/>
- Google Cloud Architecture Center on ADRs: <https://cloud.google.com/architecture/architecture-decision-records>
