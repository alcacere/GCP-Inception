# 04 · Infrastructure as Code: Terraform and Packer

The original project described its VM in code (Packer + Vagrant). This project
describes **every cloud resource** in code. The console is used for exploring and
debugging, never for creating something that has to persist.

```
infra/bootstrap  (local state, run once)
   │  creates: APIs, state bucket, WIF pool/provider, deployer SA, Artifact Registry
   ▼
infra/envs/dev   (state in gs://<project>-tfstate/envs/dev)
   │  calls modules: network, iam, secrets, database, run-service, edge, observability…
   ▼
GCP project
```

---

## Terraform

**What it is.** A declarative IaC tool. You describe the desired resources in HCL,
Terraform compares that to its **state** (what it created last time) and to reality,
then shows a **plan** of creates/updates/destroys before applying it. The `google`
provider translates resources into GCP API calls.

**Why here (and not something else).** See [ADR 0001](adr/0001-terraform-for-iac.md).
In short: it is the most requested IaC tool in job postings, it is cloud-agnostic,
and Google's own managed option (Infrastructure Manager) runs Terraform anyway.
OpenTofu is a drop-in alternative, and the code will avoid features that would break
compatibility with it.

**Core workflow**

```sh
terraform init      # download providers, configure the backend
terraform fmt       # canonical formatting (checked in CI)
terraform validate  # syntax and internal consistency
terraform plan -out=tfplan   # what would change
terraform apply tfplan       # apply exactly that plan
```

**References**
- Terraform language: <https://developer.hashicorp.com/terraform/language>
- Google provider: <https://registry.terraform.io/providers/hashicorp/google/latest/docs>
- Terraform on Google Cloud: <https://cloud.google.com/docs/terraform>
- OpenTofu: <https://opentofu.org/docs/>

---

## Remote state in Cloud Storage

**What it is.** State maps code to real resource IDs. Kept locally, it is lost with
the laptop and can't be shared with CI. The `gcs` backend stores it in a bucket and
**locks** it during operations, so two applies can't run at the same time.

**How this project will implement it.**

```hcl
terraform {
  backend "gcs" {
    bucket = "<project-id>-tfstate"
    prefix = "envs/dev"
  }
}
```

The bucket has **object versioning** (to recover a corrupted state), uniform
bucket-level access, public access prevention, and IAM restricted to the deployer SA
and my user.

> **State can contain secrets.** Some resources store sensitive attributes in
> state (for example a generated database password). That is why this project never
> lets Terraform *generate or hold* secret values. See [05](05-identity-and-security.md).

**The bootstrap problem.** The state bucket can't be stored in itself before it
exists. `infra/bootstrap` runs once with local state, then its state is migrated into
the bucket with `terraform init -migrate-state`.

**References**
- Store Terraform state in Cloud Storage: <https://cloud.google.com/docs/terraform/resource-management/store-state>
- `gcs` backend: <https://developer.hashicorp.com/terraform/language/backend/gcs>

---

## Modules and environments

**What it is.** A module is a directory of `.tf` files with **input variables** and
**outputs**. A root configuration (`envs/dev`) calls modules and wires outputs into
inputs:

```hcl
module "network" {
  source = "../../modules/network"
  region = var.region
}

module "database" {
  source          = "../../modules/database"
  network_id      = module.network.vpc_id          # dependency expressed through data
  tier            = "db-f1-micro"                  # dev: smallest; prod: larger
  deletion_protection = var.env == "prod"
}
```

**Design rules for the modules**
- One concern per module, matching the docs (network, database, edge…).
- No provider or backend blocks inside modules, only in `envs/*`.
- Every input has a `description` and a `type`, and validated where possible.
- Expensive resources use a feature flag: `count = var.enable_cache ? 1 : 0`.
- Version-pin providers in `envs/*` (`required_providers` + `.terraform.lock.hcl` committed).

**Quality tools (run in CI):** `terraform fmt -check`, `terraform validate`, `tflint`
with the Google ruleset, and an IaC security scanner (Trivy `config` or Checkov).

**References**
- Best practices for Terraform on Google Cloud: <https://cloud.google.com/docs/terraform/best-practices/general-style-structure>
- Module structure: <https://developer.hashicorp.com/terraform/language/modules/develop/structure>
- TFLint Google ruleset: <https://github.com/terraform-linters/tflint-ruleset-google>
- Google-maintained modules, for reference not for copying: <https://github.com/terraform-google-modules>

---

## Packer on GCE (Phase 1)

**What it is.** The same tool as in the original project, with a different
**builder**. `googlecompute` starts a temporary VM from a public image, runs the
provisioners over SSH, then saves the disk as a **custom image** in an **image
family**. A family always points at its newest image, so Terraform can ask for
"the latest `inception-host`" without hard-coding a name.

```
debian-12 public image ──► temp VM ──► provisioners (Docker, users, hardening) ──► image
                                                                    family: inception-host
```

**What carries over and what disappears.** The Docker installation and hardening
provisioners can be reused almost unchanged. The ISO download, the boot command and
the **preseed file disappear**, because Google's public images are already installed
systems. Cloud-init and the guest environment handle first boot instead.

**References**
- Packer `googlecompute` builder: <https://developer.hashicorp.com/packer/integrations/hashicorp/googlecompute>
- Custom images and image families: <https://cloud.google.com/compute/docs/images/image-families-best-practices>
