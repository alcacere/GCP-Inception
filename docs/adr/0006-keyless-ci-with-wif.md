# 0006 · Authenticate CI with Workload Identity Federation, never SA keys

**Status:** Accepted

## Context
GitHub Actions must run `terraform` and deploy to Cloud Run. The traditional method,
a JSON service account key stored as a GitHub secret, is a long-lived credential:
if it leaks, it works from anywhere until someone notices and revokes it.

## Decision
Use a Workload Identity Pool with a GitHub OIDC provider. The provider's attribute
condition restricts tokens to this repository. Pull-request jobs impersonate a
read-only plan SA. Only jobs on `refs/heads/main` can impersonate the deployer SA.
No service account key is ever created.

## Alternatives considered
- **SA key in GitHub secrets.** Simple, but a long-lived credential that needs manual rotation.
- **Cloud Build with its own SA.** Keyless too, but moves CI away from GitHub ([10](../10-ci-cd.md)).

## Consequences
- Tokens last about an hour and are bound to a specific workflow run.
- A misconfigured attribute condition would be a serious hole, so it is covered
  explicitly in the runbook verification steps.
- Slightly more setup in bootstrap.
