# 0007 · Use Cloud Monitoring instead of self-hosted Prometheus

**Status:** Accepted

## Context
The original stack ran Prometheus + Blackbox Exporter to probe its own endpoints. On
Cloud Run there is no always-on host for a TSDB, and probes run from inside the same
network can't see what an external user sees.

## Decision
Use Cloud Monitoring uptime checks (several regions, external), built-in metrics for
Cloud Run, Cloud SQL and the load balancer, alert policies, and an SLO with burn-rate
alerting, all defined in Terraform. Negative TLS tests (TLS 1.1 must fail) move into
`make smoke` and CI.

## Alternatives considered
- **Prometheus on a small VM.** Familiar, but adds a host to patch and pay for.
- **Managed Service for Prometheus.** Keeps PromQL. Kept as a Phase 4 item for
  continuity with the original alert rules.

## Consequences
- Nothing to operate. Probes come from outside, like a real user.
- Uptime checks can't assert that a connection *fails*, so that guarantee depends on
  the smoke test running regularly (in CI after every deploy).
- PromQL knowledge still applies: Cloud Monitoring accepts PromQL queries.
