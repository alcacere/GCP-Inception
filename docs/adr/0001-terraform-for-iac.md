# 0001 · Use Terraform for all cloud infrastructure

**Status:** Accepted

## Context
The original project used Packer + Vagrant for a local VM. On GCP, dozens of
resources (network, IAM, database, load balancer, monitoring) must be created
reproducibly, reviewed before changing, and destroyed reliably to control cost.

## Decision
All persistent GCP resources are defined in Terraform, with remote state in a
versioned Cloud Storage bucket. Packer is kept only to build the Phase 1 VM image.
The code avoids features that would break OpenTofu compatibility.

## Alternatives considered
- **`gcloud` shell scripts.** Imperative, not idempotent without a lot of extra work,
  and there is no plan/diff before changes.
- **Cloud Deployment Manager.** Google's older native tool. It has been deprecated in
  favour of Infrastructure Manager.
- **Infrastructure Manager.** Google's managed service that runs Terraform. A good
  option later, but it adds a layer before the basics are learned.
- **Pulumi.** IaC in general-purpose languages. Less common in junior job postings.

## Consequences
- A plan can be reviewed in every pull request, and the same modules serve `dev` and `prod`.
- The state must be protected: it can contain sensitive values, so secret *values*
  are kept out of Terraform ([05](../05-identity-and-security.md)).
- One more bootstrap step to create the state bucket.
