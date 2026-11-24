# 0008 · Grafana replaces Blackbox Exporter (and the Prometheus UI) in the stack

**Status:** Accepted (the IAP-JWT login is still to be validated)

## Context
[ADR 0007](0007-cloud-monitoring-over-self-hosted-prometheus.md) moved probing,
metric storage and alerting to Cloud Monitoring. That left the original Blackbox
Exporter with nothing to do. It also meant the only way to look at the platform was
the Cloud Monitoring console, which is GCP-specific. Grafana is the dashboard tool most
monitoring teams use, whichever cloud they run on, and it fits the original
project's habit of building and operating my own images.

## Decision
- Remove Blackbox Exporter and Prometheus from the stack in both phases.
- Add **Grafana**, built from my own Dockerfile. In Phase 1 it is a compose service with
  no published port, opened through an IAP tunnel. In Phase 2 it is a Cloud Run service
  at `/grafana/` on the existing load balancer, with **IAP** on the backend service.
- Data sources: Google Cloud Monitoring and Google Cloud Logging, through a
  read-only service account (`sa-grafana`).
- Data sources and dashboards are **provisioned from git** and baked into the image.
  The container keeps no state that matters.
- Alerting stays in Cloud Monitoring. Grafana is for viewing, not paging.
- Probing (the job Blackbox did) is done by Cloud Monitoring uptime checks, plus the
  negative TLS test in `make smoke`.

## Alternatives considered
- **Keep Blackbox + Prometheus on a VM.** Familiar, but adds a host, and probes from
  inside the same network miss what external users see.
- **Cloud Monitoring dashboards only.** Nothing to run, but less widely used outside
  GCP, and dashboards built in the console aren't naturally versioned.
- **Grafana Cloud (SaaS free tier).** No hosting work, but data and access live
  outside the project, and it skips the "build and operate it" learning.
- **Grafana with a Cloud SQL backend for persistent UI edits.** Not needed when
  everything is provisioned as code. Kept as an option if users other than me need
  to save dashboards.

## Consequences
- One more service to build, scan and deploy, at almost no cost (scale to zero).
- No Grafana passwords: IAP decides who gets in, and Grafana reads the identity
  from the signed header. If JWT auth doesn't work as planned, the fallback is
  anonymous read-only behind IAP + an admin password in Secret Manager.
- Changing a dashboard in the UI doesn't persist. Changes go through a pull request,
  which is the intended workflow.
- Grafana can't alert on "TLS 1.1 must fail". That guarantee stays in the smoke test.
