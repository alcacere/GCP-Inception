# 03 · GCP Foundations

Before any workload is deployed, the environment it lives in has to be set up:
the account structure, billing, which APIs are on, and where things run. In a company
this is the "landing zone", usually owned by a platform team. Here it is Phase 0.

---

## Resource hierarchy

**What it is.** Every GCP resource belongs to a **project**. Projects can sit in
**folders**, which sit under an **organization** (one per Google Workspace or Cloud
Identity domain). IAM policies and organization policies are **inherited downward**.

```
Organization (optional for a personal account)
 └── Folder (e.g. "portfolio")
      └── Project  inception-gcp-<suffix>     ← billing, APIs, IAM, quotas live here
           └── Resources (VPC, Cloud Run, Cloud SQL, buckets…)
```

**Why it matters here.** A personal account usually has no organization, so there are
no folders or org policies. That limitation is documented, not hidden. The project is
the security and billing boundary, so **one project per environment** (`dev`, later
`prod`) is the target design.

**Concepts to know.** Project ID (globally unique and permanent) vs project name vs
project number. Deleting a project has a 30-day recovery window.

**References**
- Resource hierarchy: <https://cloud.google.com/resource-manager/docs/cloud-platform-resource-hierarchy>
- Creating and managing projects: <https://cloud.google.com/resource-manager/docs/creating-managing-projects>

---

## Billing, free trial and budgets

**What it is.** A **Cloud Billing account** pays for one or more projects. New
accounts get a **free trial** (time-limited credits) and an **always-free tier** of
usage limits for some products (for example a small e2-micro VM in some US regions,
Cloud Run and Cloud Storage quotas).

**Budgets** send alerts when actual or forecasted spend crosses a threshold. They
**don't stop spending** on their own. Capping spend needs a Pub/Sub notification and
a function that disables billing, which is an optional exercise documented in
[13](13-cost-management.md).

**How this project will implement it.** A budget with 50/90/100 % alerts is the first
resource created, before any compute.

**References**
- Free program: <https://cloud.google.com/free/docs/free-cloud-features>
- Create a budget: <https://cloud.google.com/billing/docs/how-to/budgets>
- Disable billing with notifications: <https://cloud.google.com/billing/docs/how-to/disable-billing-with-notifications>

---

## APIs and services

**What it is.** Each product is an API that must be **enabled** per project before it
can be used (`run.googleapis.com`, `sqladmin.googleapis.com`, …). Enabling an API is
free. Using it is not.

**How this project will implement it.** `infra/bootstrap` enables the list with
`google_project_service`, so a fresh project is ready in one apply. The planned list:

```
compute  run  sqladmin  servicenetworking  secretmanager  artifactregistry
iam  iamcredentials  sts  cloudresourcemanager  dns  monitoring  logging
certificatemanager  iap  storage  (redis — only when the cache is enabled)
```

---

## Regions and zones

**What it is.** A **region** is a geographic area (`europe-west1`). A **zone** is an
isolated failure domain inside it (`europe-west1-b`). Resources are **zonal** (a VM,
a zonal disk), **regional** (Cloud Run, Cloud SQL HA, a regional bucket) or **global**
(the external Application Load Balancer, global VPC networks, IAM).

**Why it matters.** Location affects latency, price (regions are priced differently),
which products and machine types are available, and data residency.

**Decision.** `europe-west1` is the default (`var.region`). See the open decisions in
[PLAN.md](../PLAN.md).

**References**
- Geography and regions: <https://cloud.google.com/docs/geography-and-regions>
- Region picker tool: <https://cloud.withgoogle.com/region-picker/>

---

## Tooling: `gcloud`, Cloud Shell, and authentication

**What it is.** The `gcloud` CLI manages most resources. Cloud Shell is a free
browser shell with `gcloud` and Terraform installed. Two kinds of local login exist,
and confusing them is a common beginner mistake:

| Command | Who uses the credential | Used by |
|---|---|---|
| `gcloud auth login` | The `gcloud` CLI itself | `gcloud …` commands |
| `gcloud auth application-default login` | Client libraries (**ADC**) | Terraform, SDKs, scripts |

**How this project will implement it.** On my workstation, Terraform uses ADC with my
user account during bootstrap. Everything after bootstrap can impersonate the deployer
service account (`--impersonate-service-account`), so my day-to-day permissions match
what CI has.

**References**
- gcloud CLI overview: <https://cloud.google.com/sdk/gcloud>
- Application Default Credentials: <https://cloud.google.com/docs/authentication/application-default-credentials>
- Service account impersonation: <https://cloud.google.com/docs/authentication/use-service-account-impersonation>

---

## Labels and naming

Every resource will carry labels: `project=inception`, `env=dev`, `phase=<n>`,
`managed_by=terraform`. Billing exports and the cost report can group by label, which
is how the per-phase cost comparison in [13](13-cost-management.md) is produced.

Naming convention: `inception-<env>-<component>` (for example `inception-dev-sql`).
Bucket names are global, so they get the project ID as a prefix.

**References**
- Labels: <https://cloud.google.com/resource-manager/docs/labels-overview>
- Naming best practices (Terraform on GCP): <https://cloud.google.com/docs/terraform/best-practices/general-style-structure>
