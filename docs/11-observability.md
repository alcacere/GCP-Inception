# 11 · Observability: Cloud Logging, Cloud Monitoring, SLOs, Grafana

## Why it changes and what stays the same

The original stack monitored itself **from the outside, like a user**: Blackbox
Exporter probed HTTPS, TLS versions, certificate expiry and TCP ports, and Prometheus
stored the results and evaluated alerts. That idea stays. The tools become managed
services ([ADR 0007](adr/0007-cloud-monitoring-over-self-hosted-prometheus.md)), and
**Grafana takes Blackbox's place in the stack** as the one container I build and run
for monitoring ([ADR 0008](adr/0008-grafana-replaces-blackbox.md)).

The split of responsibilities:

```
  Uptime checks (several regions) ──────┐
  Cloud Run / Cloud SQL / LB metrics ───┼──► Cloud Monitoring ──► alert policies, SLOs ──► email
  Logs (containers, LB, audit) ─────────┘          │
                                                   ▼ read-only API (sa-grafana)
                                     Grafana on Cloud Run, behind IAP
                                     dashboards + data sources from git
```

Cloud Monitoring **collects, probes and alerts**. Grafana **shows**. Grafana doesn't
probe anything itself, so the Blackbox probing job goes to uptime checks.

| Original | GCP | Notes |
|---|---|---|
| Blackbox `https_2xx` probe | **Uptime check** (HTTPS) | Runs from several regions, not from inside the same network |
| Certificate expiry from the probe | Uptime check SSL certificate validation + an expiry alert | |
| Blackbox `tls_old` negative test | `make smoke` (`openssl s_client -tls1_1` must fail) | Uptime checks can't assert that something *fails* |
| Blackbox `tcp_connect` probes | Not needed. Cloud SQL and Memorystore have no public port and report their own health metrics | |
| Prometheus TSDB | Cloud Monitoring metrics | System metrics for Cloud Run, Cloud SQL and the LB are automatic |
| Prometheus web UI at `/prometheus/` | **Grafana** at `/grafana/`, behind IAP | Same single-entry-point idea, but identity-gated instead of open |
| `alerts.yml` | **Alert policies** (Terraform) | Notification channel: email (Slack optional) |
| PromQL | Monitoring Query Language is deprecated. **PromQL is supported in Cloud Monitoring** | My original queries are still useful |
| `docker logs` | **Cloud Logging** | Container stdout/stderr, LB request logs, audit logs |

---

## Cloud Logging

**What it is.** Central log storage. Every Cloud Run container's stdout/stderr, every
load balancer request (if logging is on for the backend service), every Cloud SQL
error log, and every admin API call (audit logs) end up here. Logs are queried with
the Logging query language and routed by **sinks** to buckets, BigQuery or Pub/Sub.

**How this project will implement it**
- Containers log **structured JSON** to stdout where possible (NGINX `log_format`
  with `escape=json`), so fields like `status` and `latency` can be queried.
- LB request logging on, with a sample rate to control cost.
- **Log-based metrics** for things that aren't metrics yet, for example the count of
  Cloud Armor denials or WordPress PHP fatal errors.
- Retention: the default bucket retention. Longer retention costs money.

```text
# Example queries
resource.type="cloud_run_revision" AND resource.labels.service_name="wordpress" AND severity>=ERROR
resource.type="http_load_balancer" AND jsonPayload.enforcedSecurityPolicy.outcome="DENY"
protoPayload.serviceName="secretmanager.googleapis.com"   # who read which secret
```

**References**
- Cloud Logging overview: <https://cloud.google.com/logging/docs/overview>
- Logging query language: <https://cloud.google.com/logging/docs/view/logging-query-language>
- Log-based metrics: <https://cloud.google.com/logging/docs/logs-based-metrics>
- Structured logging: <https://cloud.google.com/logging/docs/structured-logging>

---

## Cloud Monitoring: uptime checks and alerting

**Uptime checks** send HTTP(S)/TCP requests to a public endpoint from several regions
around the world, and record availability and latency.

**Planned checks**

| Check | Target | Asserts |
|---|---|---|
| `site-https` | `https://${DOMAIN}/` | 200, content contains the site title, valid certificate |
| `site-health` | `https://${DOMAIN}/healthz` | 200 (the NGINX endpoint from the original project, independent of PHP) |
| `portfolio` | `https://${DOMAIN}/portfolio/` | 200 from the CDN |

**Planned alert policies** (all in Terraform, `modules/observability`)

| Alert | Condition | Original equivalent |
|---|---|---|
| Site down | Uptime check failing from ≥ 2 regions for 2 min | `probe_success == 0` |
| Certificate expiring | Less than 14 days until expiry | `probe_ssl_earliest_cert_expiry` |
| 5xx rate | LB 5xx ratio > 5 % for 5 min | — (new) |
| Cloud SQL resources | CPU > 80 % or disk > 85 % for 10 min | — (new) |
| SLO burn rate | Fast burn (see below) | — (new) |
| Budget | (Billing, see [13](13-cost-management.md)) | — |

**References**
- Uptime checks: <https://cloud.google.com/monitoring/uptime-checks>
- Alerting overview: <https://cloud.google.com/monitoring/alerts>
- PromQL in Cloud Monitoring: <https://cloud.google.com/monitoring/promql>
- Cloud Run metrics: <https://cloud.google.com/run/docs/monitoring>

---

## SLOs and error budgets

**What it is.** A **Service Level Indicator** (SLI) is a measurement, such as "the
fraction of requests that return non-5xx in under 1 s". A **Service Level Objective**
(SLO) is the target for it (99 % over 28 days). The **error budget** is the remaining
1 %. **Burn-rate alerts** fire when the budget is being used up too quickly, which is
more meaningful than alerting on every blip.

**How this project will implement it.** Cloud Monitoring *service monitoring* on the
Cloud Run service:

- Availability SLO: 99 % of requests are not 5xx, over a rolling 28 days.
- Latency SLO: 95 % of requests are faster than 1000 ms.
- Alert when the burn rate is > 10× over 1 hour (fast burn).

The targets are deliberately modest: a shared-core database and scale-to-zero can't
promise more, and *choosing a realistic SLO* is the point of the exercise.

**References**
- Service monitoring and SLOs: <https://cloud.google.com/stackdriver/docs/solutions/slo-monitoring>
- Google SRE workbook, implementing SLOs: <https://sre.google/workbook/implementing-slos/>
- Alerting on SLOs: <https://sre.google/workbook/alerting-on-slos/>

---

## Grafana: dashboards as code

**What it is.** An open-source visualization tool. It doesn't store metrics itself: it
queries **data sources** (Prometheus, Cloud Monitoring, Loki, SQL databases…) and
renders **dashboards** of panels. It can also alert, but here alerting stays in Cloud
Monitoring so there is one source of truth for pages.

**Why here.** Grafana is the dashboard tool most monitoring teams use, whatever cloud
they run on, so it carries over to other jobs better than the Cloud Monitoring
console. It also keeps the original project's "build it, don't pull it" spirit: it
is the one monitoring component I package and operate myself.

**How this project will implement it**

| Concern | Decision |
|---|---|
| Image | Built from my own Dockerfile (pinned Grafana release, the same supply-chain rules as the other images), pushed to Artifact Registry |
| Where it runs | Phase 1: a container in the compose stack, port 3000 not published, opened with an IAP tunnel. Phase 2: Cloud Run service `grafana`, `max-instances=1`, `min-instances=0` |
| Exposure | Path `/grafana/` on the existing load balancer (`serve_from_sub_path`), so no extra forwarding rule or certificate. The backend service has **IAP** on. Cloud Run ingress is `internal-and-cloud-load-balancing` |
| Login | Planned: Grafana's **JWT auth** reads the IAP header (`X-Goog-IAP-JWT-Assertion`) and validates it against Google's public keys, so the IAP identity *is* the Grafana user. No Grafana passwords exist. To validate during implementation. Fallback: anonymous viewer behind IAP + an admin password in Secret Manager |
| Data sources | **Google Cloud Monitoring** (metrics, SLOs) and **Google Cloud Logging**, authenticated with the attached service account `sa-grafana` (`monitoring.viewer`, `logging.viewer`). No key file |
| State | **Provisioned from git**: `srcs/requirements/grafana/provisioning/{datasources,dashboards}/` is baked into the image. The internal SQLite database holds nothing that matters, so the container can be stateless. UI edits are lost on restart *on purpose*: a dashboard change is a pull request |
| Hardening | `allow_sign_up=false`, analytics/reporting off, plugin installs pinned at build time, runs as a non-root user |

**Planned dashboards (JSON in git)**

| Dashboard | Panels |
|---|---|
| Platform overview | Uptime check status per region, certificate days left, LB requests and 5xx ratio, p50/p95 latency |
| Cloud Run | Instance count, cold starts, CPU/memory per container (nginx vs php-fpm), revision in service |
| Cloud SQL | CPU, memory, connections, disk usage, replication lag (if HA) |
| SLO | SLI over 28 days, remaining error budget, burn rate |
| Security | Cloud Armor allow/deny counts by rule (log-based metric), IAP access denials, secret access events |
| Cost proxy | Cloud Run billable instance time, LB data processed (tie-in with [13](13-cost-management.md)) |

**References**
- Grafana documentation: <https://grafana.com/docs/grafana/latest/>
- Provisioning (data sources and dashboards as code): <https://grafana.com/docs/grafana/latest/administration/provisioning/>
- Google Cloud Monitoring data source: <https://grafana.com/docs/grafana/latest/datasources/google-cloud-monitoring/>
- Google Cloud Logging data source: <https://grafana.com/grafana/plugins/googlecloud-logging-datasource/>
- JWT authentication: <https://grafana.com/docs/grafana/latest/setup-grafana/configure-security/configure-authentication/jwt/>
- Run behind a reverse proxy on a sub-path: <https://grafana.com/tutorials/run-grafana-behind-a-proxy/>
- Configure Grafana in Docker: <https://grafana.com/docs/grafana/latest/setup-grafana/configure-docker/>
- IAP signed headers: <https://cloud.google.com/iap/docs/signed-headers-howto>
- IAP for Cloud Run: <https://cloud.google.com/iap/docs/enabling-cloud-run>

---

## Optional: Managed Service for Prometheus

To show continuity with the original project, Google Cloud Managed Service for
Prometheus can ingest Prometheus metrics and evaluate the original PromQL rules. On
Cloud Run this needs a collector sidecar. Grafana can query it with its Prometheus
data source, so the original PromQL panels would work unchanged. It is a Phase 4 item.

**References**
- Managed Service for Prometheus: <https://cloud.google.com/stackdriver/docs/managed-prometheus>
