# 0002 · Migrate in phases: rehost on Compute Engine, then replatform

**Status:** Accepted

## Context
The goal is a portfolio project that demonstrates cloud engineering skills. Going
straight to Cloud Run would skip VM, disk, image and firewall work, which is core
certification and interview material. It would also leave no baseline to compare
against.

## Decision
Phase 1 runs the original Docker Compose stack, unchanged, on a Compute Engine VM
built with Packer and provisioned with Terraform. Phase 2 replaces components with
managed services. Phase 1 stays up until Phase 2 is verified, which gives a rollback
path for the data migration.

## Alternatives considered
- **Straight to Cloud Run.** Faster, but loses the IaaS practice and the comparison.
- **Straight to GKE.** Too much complexity and cost for one application.

## Consequences
- More total work, but each phase is a complete, demonstrable result.
- A measured comparison (cost, deploy time, rollback time, patching effort) becomes a
  project deliverable ([08](../08-compute-and-containers.md)).
- Two sets of resources briefly coexist during the cut-over, which costs extra for a few days.
