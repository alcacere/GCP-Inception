# Sources

Every reference used in this documentation, grouped by topic. All links were checked
on 2026-11-23. Official documentation is preferred over blog posts. When the two
disagree, the official page wins.

## The original project
- Inception (42 curriculum version): <https://github.com/alcacere/inception>

## Architecture and methodology
- Google Cloud Well-Architected Framework: <https://cloud.google.com/architecture/framework>
- Cloud Architecture Center: <https://cloud.google.com/architecture>
- Migrate to Google Cloud, getting started: <https://cloud.google.com/architecture/migration-to-gcp-getting-started>
- Security foundations blueprint: <https://cloud.google.com/architecture/security-foundations>
- Architecture decision records (Google): <https://cloud.google.com/architecture/architecture-decision-records>
- ADR templates and tooling: <https://adr.github.io/>
- *Documenting Architecture Decisions*, Michael Nygard: <https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions>

## Foundations
- Resource hierarchy: <https://cloud.google.com/resource-manager/docs/cloud-platform-resource-hierarchy>
- Creating and managing projects: <https://cloud.google.com/resource-manager/docs/creating-managing-projects>
- Labels: <https://cloud.google.com/resource-manager/docs/labels-overview>
- Organization policy constraints: <https://cloud.google.com/resource-manager/docs/organization-policy/org-policy-constraints>
- Geography and regions: <https://cloud.google.com/docs/geography-and-regions>
- Region picker: <https://cloud.withgoogle.com/region-picker/>
- gcloud CLI: <https://cloud.google.com/sdk/gcloud>
- Application Default Credentials: <https://cloud.google.com/docs/authentication/application-default-credentials>
- Service account impersonation: <https://cloud.google.com/docs/authentication/use-service-account-impersonation>

## Infrastructure as Code
- Terraform language: <https://developer.hashicorp.com/terraform/language>
- Google provider: <https://registry.terraform.io/providers/hashicorp/google/latest/docs>
- Terraform on Google Cloud: <https://cloud.google.com/docs/terraform>
- Terraform best practices on Google Cloud: <https://cloud.google.com/docs/terraform/best-practices/general-style-structure>
- Store state in Cloud Storage: <https://cloud.google.com/docs/terraform/resource-management/store-state>
- `gcs` backend: <https://developer.hashicorp.com/terraform/language/backend/gcs>
- Module structure: <https://developer.hashicorp.com/terraform/language/modules/develop/structure>
- TFLint Google ruleset: <https://github.com/terraform-linters/tflint-ruleset-google>
- Google-maintained modules: <https://github.com/terraform-google-modules>
- OpenTofu: <https://opentofu.org/docs/>
- Packer `googlecompute`: <https://developer.hashicorp.com/packer/integrations/hashicorp/googlecompute>
- Image families: <https://cloud.google.com/compute/docs/images/image-families-best-practices>

## Identity and security
- IAM overview: <https://cloud.google.com/iam/docs/overview>
- Roles: <https://cloud.google.com/iam/docs/understanding-roles>
- Using IAM securely: <https://cloud.google.com/iam/docs/using-iam-securely>
- Service accounts: <https://cloud.google.com/iam/docs/service-account-overview>
- Service account best practices: <https://cloud.google.com/iam/docs/best-practices-service-accounts>
- Workload Identity Federation: <https://cloud.google.com/iam/docs/workload-identity-federation>
- WIF with deployment pipelines: <https://cloud.google.com/iam/docs/workload-identity-federation-with-deployment-pipelines>
- Secret Manager: <https://cloud.google.com/secret-manager/docs/overview>
- Secret Manager best practices: <https://cloud.google.com/secret-manager/docs/best-practices>
- Identity-Aware Proxy: <https://cloud.google.com/iap/docs/concepts-overview>
- IAP TCP forwarding: <https://cloud.google.com/iap/docs/using-tcp-forwarding>
- Cloud Audit Logs: <https://cloud.google.com/logging/docs/audit>

## Networking
- VPC: <https://cloud.google.com/vpc/docs/vpc>
- Subnets: <https://cloud.google.com/vpc/docs/subnets>
- Firewall rules: <https://cloud.google.com/firewall/docs/firewalls>
- Cloud NAT: <https://cloud.google.com/nat/docs/overview>
- Private services access: <https://cloud.google.com/vpc/docs/private-services-access>
- Cloud DNS: <https://cloud.google.com/dns/docs/overview>
- Cloud Run Direct VPC egress: <https://cloud.google.com/run/docs/configuring/vpc-direct-vpc>
- Cloud Run VPC options compared: <https://cloud.google.com/run/docs/configuring/connecting-vpc>

## Edge
- Application Load Balancer: <https://cloud.google.com/load-balancing/docs/https>
- Serverless NEGs: <https://cloud.google.com/load-balancing/docs/negs/serverless-neg-concepts>
- LB with Cloud Run: <https://cloud.google.com/load-balancing/docs/https/setup-global-ext-https-serverless>
- URL maps: <https://cloud.google.com/load-balancing/docs/url-map-concepts>
- Backend buckets: <https://cloud.google.com/load-balancing/docs/backend-bucket>
- Health check ranges: <https://cloud.google.com/load-balancing/docs/health-check-concepts#ip-ranges>
- Google-managed certificates: <https://cloud.google.com/load-balancing/docs/ssl-certificates/google-managed-certs>
- Certificate Manager: <https://cloud.google.com/certificate-manager/docs/overview>
- DNS authorizations: <https://cloud.google.com/certificate-manager/docs/dns-authorizations>
- SSL policies: <https://cloud.google.com/load-balancing/docs/ssl-policies-concepts>
- Cloud Armor: <https://cloud.google.com/armor/docs/cloud-armor-overview>
- Cloud Armor WAF rules: <https://cloud.google.com/armor/docs/waf-rules>
- Cloud Armor rate limiting: <https://cloud.google.com/armor/docs/rate-limiting-overview>
- Cloud CDN: <https://cloud.google.com/cdn/docs/overview>
- Static website on Cloud Storage: <https://cloud.google.com/storage/docs/hosting-static-website>

## Compute and containers
- Artifact Registry: <https://cloud.google.com/artifact-registry/docs/overview>
- Cleanup policies: <https://cloud.google.com/artifact-registry/docs/repositories/cleanup-policy>
- Artifact Analysis scanning: <https://cloud.google.com/artifact-analysis/docs/container-scanning-overview>
- Compute Engine: <https://cloud.google.com/compute/docs/overview>
- Machine families: <https://cloud.google.com/compute/docs/machine-resource>
- Scheduled snapshots: <https://cloud.google.com/compute/docs/disks/scheduled-snapshots>
- Spot VMs: <https://cloud.google.com/compute/docs/instances/spot>
- Ops Agent: <https://cloud.google.com/stackdriver/docs/solutions/agents/ops-agent>
- What is Cloud Run: <https://cloud.google.com/run/docs/overview/what-is-cloud-run>
- Container runtime contract: <https://cloud.google.com/run/docs/container-contract>
- Sidecars: <https://cloud.google.com/run/docs/deploying#sidecars>
- Cloud Storage volume mounts: <https://cloud.google.com/run/docs/configuring/services/cloud-storage-volume-mounts>
- Cloud Run secrets: <https://cloud.google.com/run/docs/configuring/services/secrets>
- Cloud Run ingress: <https://cloud.google.com/run/docs/securing/ingress>
- Rollouts and rollbacks: <https://cloud.google.com/run/docs/rollouts-rollbacks-traffic-migration>
- GKE Autopilot: <https://cloud.google.com/kubernetes-engine/docs/concepts/autopilot-overview>

## Data
- Cloud SQL for MySQL: <https://cloud.google.com/sql/docs/mysql>
- Editions: <https://cloud.google.com/sql/docs/mysql/editions-intro>
- Private IP: <https://cloud.google.com/sql/docs/mysql/private-ip>
- Private Service Connect: <https://cloud.google.com/sql/docs/mysql/about-private-service-connect>
- Connect from Cloud Run: <https://cloud.google.com/sql/docs/mysql/connect-run>
- Backups: <https://cloud.google.com/sql/docs/mysql/backup-recovery/backups>
- Point-in-time recovery: <https://cloud.google.com/sql/docs/mysql/backup-recovery/pitr>
- High availability: <https://cloud.google.com/sql/docs/mysql/high-availability>
- IAM database authentication: <https://cloud.google.com/sql/docs/mysql/iam-authentication>
- SQL import/export: <https://cloud.google.com/sql/docs/mysql/import-export/import-export-sql>
- Cloud Storage: <https://cloud.google.com/storage/docs/introduction>
- Storage classes: <https://cloud.google.com/storage/docs/storage-classes>
- Object versioning: <https://cloud.google.com/storage/docs/object-versioning>
- Soft delete: <https://cloud.google.com/storage/docs/soft-delete>
- Lifecycle management: <https://cloud.google.com/storage/docs/lifecycle>
- Memorystore for Redis: <https://cloud.google.com/memorystore/docs/redis>
- Memorystore for Valkey: <https://cloud.google.com/memorystore/docs/valkey>

## CI/CD and supply chain
- GitHub OIDC with Google Cloud: <https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/configuring-openid-connect-in-google-cloud-platform>
- GitHub Actions security hardening: <https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions>
- GitHub Environments: <https://docs.github.com/en/actions/managing-workflow-runs-and-deployments/managing-deployments/managing-environments-for-deployment>
- `google-github-actions/auth`: <https://github.com/google-github-actions/auth>
- `google-github-actions/deploy-cloudrun`: <https://github.com/google-github-actions/deploy-cloudrun>
- Cloud Build: <https://cloud.google.com/build/docs/overview>
- Cloud Build → Cloud Run: <https://cloud.google.com/build/docs/deploying-builds/deploy-cloud-run>
- Binary Authorization: <https://cloud.google.com/binary-authorization/docs/overview>
- Trivy: <https://trivy.dev/latest/docs/>
- hadolint: <https://github.com/hadolint/hadolint>
- ShellCheck: <https://www.shellcheck.net/>
- gitleaks: <https://github.com/gitleaks/gitleaks>
- SLSA: <https://slsa.dev/>

## Observability
- Cloud Logging: <https://cloud.google.com/logging/docs/overview>
- Logging query language: <https://cloud.google.com/logging/docs/view/logging-query-language>
- Log-based metrics: <https://cloud.google.com/logging/docs/logs-based-metrics>
- Structured logging: <https://cloud.google.com/logging/docs/structured-logging>
- Uptime checks: <https://cloud.google.com/monitoring/uptime-checks>
- Alerting: <https://cloud.google.com/monitoring/alerts>
- PromQL in Cloud Monitoring: <https://cloud.google.com/monitoring/promql>
- Cloud Run monitoring: <https://cloud.google.com/run/docs/monitoring>
- SLO monitoring: <https://cloud.google.com/stackdriver/docs/solutions/slo-monitoring>
- Managed Service for Prometheus: <https://cloud.google.com/stackdriver/docs/managed-prometheus>
- Grafana documentation: <https://grafana.com/docs/grafana/latest/>
- Grafana provisioning: <https://grafana.com/docs/grafana/latest/administration/provisioning/>
- Grafana Google Cloud Monitoring data source: <https://grafana.com/docs/grafana/latest/datasources/google-cloud-monitoring/>
- Grafana Google Cloud Logging data source: <https://grafana.com/grafana/plugins/googlecloud-logging-datasource/>
- Grafana JWT authentication: <https://grafana.com/docs/grafana/latest/setup-grafana/configure-security/configure-authentication/jwt/>
- Grafana behind a reverse proxy: <https://grafana.com/tutorials/run-grafana-behind-a-proxy/>
- Grafana in Docker: <https://grafana.com/docs/grafana/latest/setup-grafana/configure-docker/>
- IAP signed headers: <https://cloud.google.com/iap/docs/signed-headers-howto>
- IAP for Cloud Run: <https://cloud.google.com/iap/docs/enabling-cloud-run>
- SRE workbook, implementing SLOs: <https://sre.google/workbook/implementing-slos/>
- SRE workbook, alerting on SLOs: <https://sre.google/workbook/alerting-on-slos/>

## Cost
- Free program and free tier: <https://cloud.google.com/free/docs/free-cloud-features>
- Budgets: <https://cloud.google.com/billing/docs/how-to/budgets>
- Programmatic budget notifications: <https://cloud.google.com/billing/docs/how-to/budgets-programmatic-notifications>
- Disable billing with notifications: <https://cloud.google.com/billing/docs/how-to/disable-billing-with-notifications>
- Billing export to BigQuery: <https://cloud.google.com/billing/docs/how-to/export-data-bigquery>
- Pricing calculator: <https://cloud.google.com/products/calculator>
- Cloud Run pricing: <https://cloud.google.com/run/pricing>
- Cloud SQL pricing: <https://cloud.google.com/sql/pricing>
- Load balancing pricing: <https://cloud.google.com/vpc/network-pricing#lb>
- Cloud Armor pricing: <https://cloud.google.com/armor/pricing>
- Memorystore pricing: <https://cloud.google.com/memorystore/docs/redis/pricing>
- Cost optimization pillar: <https://cloud.google.com/architecture/framework/cost-optimization>
- FinOps Foundation framework: <https://www.finops.org/framework/>

## Learning and certification
- Associate Cloud Engineer: <https://cloud.google.com/learn/certification/cloud-engineer>
- Google Cloud Skills Boost: <https://www.cloudskillsboost.google/>
