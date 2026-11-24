# 13 · Cost Management (FinOps)

A portfolio project on a personal card has to be **cheap by design** and
**impossible to forget about**. Showing cost awareness is also one of the clearest
signals a junior cloud candidate can give.

---

## Guardrails, in the order they are created

1. **Budget + alerts** (50 / 90 / 100 % of a small monthly amount, plus forecasted
   spend) on the billing account, scoped to the project. This comes before any other
   resource.
2. **Labels on every resource** (`project`, `env`, `phase`), so cost is attributable.
3. **Feature flags for expensive resources** (`enable_cache`, `enable_cdn`,
   `enable_armor`, `min_instances`).
4. **`max-instances` on Cloud Run**, a hard ceiling on scaling costs.
5. **`make destroy`**, tested from Phase 0 onward.
6. *(Optional exercise)* Budget → Pub/Sub → Cloud Run function that **disables
   billing** on the project at 100 %. It works as a real kill switch, but it stops
   everything, including deletion protection timers. Documented and tested once.

**References**
- Budgets and alerts: <https://cloud.google.com/billing/docs/how-to/budgets>
- Programmatic budget notifications: <https://cloud.google.com/billing/docs/how-to/budgets-programmatic-notifications>
- Cost optimization pillar (Well-Architected Framework): <https://cloud.google.com/architecture/framework/cost-optimization>

---

## Where the money goes

> Prices vary by region and change over time. The **Pricing Calculator** estimate is
> the source of truth and gets recorded in this table during implementation. The
> "driver" column is what matters for design.

| Resource | Cost driver | Relative weight | Control |
|---|---|---|---|
| Compute Engine VM (Phase 1) | Machine type × hours, disk size | Medium | Right-size, stop when not demoing, Spot for dev |
| Static external IP | Charged per hour, more when *unused* | Low | Release it on destroy |
| Cloud SQL | Instance tier × hours (it runs 24/7, **no scale to zero**), storage, backups | Medium | Shared-core tier in dev, stop the instance when idle |
| External Application LB | **Each forwarding rule is charged per hour**, plus data processed | Medium (a fixed floor) | One forwarding rule (no `:80` rule) |
| Cloud Armor | Per policy + per rule + per request | Low–medium | Few rules. Disable the flag in dev if needed |
| Memorystore | Capacity × hours, always on | **High** relative to everything else | Off by default |
| Cloud Run | vCPU/memory-seconds while serving, requests | Very low at portfolio traffic (free tier) | `min-instances=0`, `max-instances=3` |
| Grafana on Cloud Run | Instance time only while someone is looking at it, plus Cloud Monitoring read API calls | Very low | `min-instances=0`, `max-instances=1`, sensible dashboard refresh intervals (not 5 s) |
| IAP | Free for HTTPS resources | — | — |
| Cloud Storage, Artifact Registry | GB stored, operations, egress | Very low | Lifecycle and cleanup policies |
| Secret Manager | Active versions + access operations | Very low | Disable/destroy old versions |
| Cloud Logging / Monitoring | Ingestion above the free allotment | Low | Sample LB logs, exclusion filters for noisy logs |
| Network egress | GB to the internet | Low | CDN for static files |

**Key insight to document:** the managed Phase 2 architecture is *not automatically
cheaper* than one small VM. The load balancer and Cloud SQL have fixed hourly floors,
while a VM running everything is a single bill. Phase 2 trades money for less
operational work (no OS patching, backups, scaling, HA). Measuring that trade-off
honestly is part of the project. See the comparison table in [08](08-compute-and-containers.md).

**References**
- Pricing calculator: <https://cloud.google.com/products/calculator>
- Cloud Run pricing: <https://cloud.google.com/run/pricing>
- Cloud SQL pricing: <https://cloud.google.com/sql/pricing>
- Load balancing pricing: <https://cloud.google.com/vpc/network-pricing#lb>
- Cloud Armor pricing: <https://cloud.google.com/armor/pricing>
- Memorystore pricing: <https://cloud.google.com/memorystore/docs/redis/pricing>
- Free tier usage limits: <https://cloud.google.com/free/docs/free-cloud-features#free-tier-usage-limits>

---

## Working rhythm to keep the bill small

- Build and test each phase **inside a time-boxed session**, then `make destroy`.
  Infrastructure as code makes rebuilding cheap. That is the point of it.
- Keep a "demo mode" (`make demo-up`) that turns on `min-instances=1` and the cache
  for an interview, and a "dev mode" that is as small as possible.
- Review the billing report **weekly**, grouped by SKU, during active development.
- Export billing data to BigQuery (optional) for SQL queries over cost. It is a good
  small data exercise and a common FinOps practice.

**References**
- Export billing to BigQuery: <https://cloud.google.com/billing/docs/how-to/export-data-bigquery>
- FinOps Foundation framework: <https://www.finops.org/framework/>
