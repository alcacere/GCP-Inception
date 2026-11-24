# 10 · CI/CD

The original project's roadmap listed "CI pipeline: ShellCheck, hadolint, compose
validation, image builds, an end-to-end smoke test" and "publish versioned images to
a registry, and deploy by digest". This page is that roadmap item, on GCP.

---

## Pipeline design

```
Pull request ──► ci.yml
  ├── lint        shellcheck · hadolint · terraform fmt -check · tflint
  ├── security    trivy config (IaC) · gitleaks (secrets in the diff)
  ├── build       docker build every image (no push)
  ├── scan        trivy image --severity HIGH,CRITICAL --exit-code 1
  └── plan        WIF auth (read-only SA) → terraform plan → comment on the PR

Merge to main ──► deploy.yml
  ├── build+push  images tagged with the git SHA → Artifact Registry → record the digests
  ├── apply       terraform apply (GitHub Environment "dev": requires my approval)
  ├── deploy      gcloud run deploy --image …@sha256:<digest> --no-traffic
  ├── smoke       make smoke against the new revision's tagged URL
  ├── promote     move 100 % of traffic to the new revision
  └── on failure  traffic stays on the previous revision → alert
```

**Principles**
- **The same artifact moves forward.** Images are built once and promoted by digest,
  never rebuilt for each environment.
- **Plan in pull requests, apply on merge.** Every infrastructure change is reviewed
  as a plan before it runs.
- **Two CI identities.** The pull-request job can only *read* (plan). Only `main` can
  apply. This is enforced by the WIF attribute condition on the ref, not just by the
  workflow file.
- **Deploy with no traffic, test, then promote.** This is a cheap blue/green pattern,
  using Cloud Run revision tags.

---

## GitHub Actions + Workload Identity Federation

**What it is.** GitHub's built-in CI. With `permissions: id-token: write`, a job can
request an OIDC token, which `google-github-actions/auth` exchanges through WIF.

```yaml
permissions:
  contents: read
  id-token: write
steps:
  - uses: actions/checkout@v4
  - uses: google-github-actions/auth@v2
    with:
      workload_identity_provider: ${{ vars.WIF_PROVIDER }}
      service_account: ${{ vars.DEPLOYER_SA }}
  - uses: google-github-actions/setup-gcloud@v2
```

The provider and service account names are not secrets (they are useless without a
matching OIDC token), so they are stored as repository **variables**, not secrets.

**Third-party action pinning.** Actions are pinned to a full commit SHA in the final
version, because a tag can be moved by whoever controls the action's repository.

**References**
- GitHub OIDC with Google Cloud: <https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/configuring-openid-connect-in-google-cloud-platform>
- `google-github-actions/auth`: <https://github.com/google-github-actions/auth>
- `google-github-actions/deploy-cloudrun`: <https://github.com/google-github-actions/deploy-cloudrun>
- GitHub Environments: <https://docs.github.com/en/actions/managing-workflow-runs-and-deployments/managing-deployments/managing-environments-for-deployment>
- Security hardening for GitHub Actions: <https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions>

---

## Alternative: Cloud Build

**What it is.** Google's managed CI service. Builds run as a list of container steps
in `cloudbuild.yaml`, triggered by repository events, with a service account identity
inside the project. No federation is needed.

**Why it isn't the primary choice.** GitHub Actions is more common in job postings
and keeps the CI definition next to the code. Cloud Build is documented here because
it shows up in the certification and in GCP-centric teams. A `cloudbuild.yaml` that
builds and pushes the images is a small optional exercise.

**References**
- Cloud Build overview: <https://cloud.google.com/build/docs/overview>
- Deploy to Cloud Run with Cloud Build: <https://cloud.google.com/build/docs/deploying-builds/deploy-cloud-run>

---

## Scanning and supply chain

| Check | Tool | Catches |
|---|---|---|
| Shell scripts | ShellCheck | Quoting bugs, POSIX issues |
| Dockerfiles | hadolint | Unpinned packages, missing `--no-cache`, root user |
| Terraform | `fmt`, `validate`, tflint | Style, typos, invalid machine types and regions |
| IaC misconfigurations | Trivy `config` (or Checkov) | Public buckets, SQL with a public IP, missing encryption or logging |
| Committed secrets | gitleaks | Keys or passwords in a diff |
| Image CVEs | Trivy `image` | Vulnerable OS packages and libraries |
| (stretch) Provenance | Binary Authorization + attestations | Deploying an image CI didn't build |

**References**
- Trivy: <https://trivy.dev/latest/docs/>
- hadolint: <https://github.com/hadolint/hadolint>
- ShellCheck: <https://www.shellcheck.net/>
- gitleaks: <https://github.com/gitleaks/gitleaks>
- SLSA framework: <https://slsa.dev/>
- Binary Authorization: <https://cloud.google.com/binary-authorization/docs/overview>
