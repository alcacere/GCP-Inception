# 08 · Compute and Containers

Everything in the original docs about container fundamentals still applies: images and
layers, PID 1, foreground processes, the entrypoint pattern. What changes is **who
schedules the containers** and **where the images live**.

---

## Artifact Registry

**What it is.** Google's managed package registry (Docker images, plus language
packages). It replaces the deprecated Container Registry. Repositories are regional
and IAM-controlled.

**How this project will implement it**
- One Docker repository: `europe-west1-docker.pkg.dev/<project>/inception`.
- Images are tagged with the **git SHA** (and `latest` only for convenience) and
  **deployed by digest** (`…/wordpress@sha256:…`), so what runs is exactly what CI
  scanned.
- **Cleanup policies** keep the last N versions, which keeps storage costs flat.
- **Vulnerability scanning** (Artifact Analysis) is optional and paid. Trivy in CI is
  the free baseline.

**References**
- Artifact Registry overview: <https://cloud.google.com/artifact-registry/docs/overview>
- Cleanup policies: <https://cloud.google.com/artifact-registry/docs/repositories/cleanup-policy>
- Artifact Analysis scanning: <https://cloud.google.com/artifact-analysis/docs/container-scanning-overview>

---

## Compute Engine (Phase 1)

**What it is.** IaaS virtual machines. You choose the machine type, the boot image,
disks and network, and you are responsible for everything inside the guest OS.

**How this project will implement it**

| Setting | Value | Why |
|---|---|---|
| Machine type | `e2-small` or `e2-medium` | Nine containers don't fit in an `e2-micro` (1 GB RAM). Measure, then right-size |
| Boot image | `family/inception-host` (Packer) | Golden image, the same idea as the original |
| Data disk | Separate persistent disk mounted at `/data` | Data survives VM recreation, and snapshots are disk-level |
| Snapshot schedule | Daily, 7-day retention | A cheap, crash-consistent backup |
| Service account | `sa-host` | Never the default compute SA |
| Shielded VM, OS Login | On | Boot integrity, IAM-managed SSH |
| Startup script | Fetch secrets → tmpfs, `make up` | The cloud version of `vagrant up` + `make` |
| Agent | Ops Agent | Logs and host metrics into Cloud Monitoring |

**Spot VMs** (much cheaper, can be stopped at any time) are a good fit for a dev or
demo host. The trade-off is documented, but not used for anything that must stay up.

**References**
- Compute Engine overview: <https://cloud.google.com/compute/docs/overview>
- Machine families: <https://cloud.google.com/compute/docs/machine-resource>
- Snapshot schedules: <https://cloud.google.com/compute/docs/disks/scheduled-snapshots>
- Ops Agent: <https://cloud.google.com/stackdriver/docs/solutions/agents/ops-agent>
- Spot VMs: <https://cloud.google.com/compute/docs/instances/spot>

---

## Cloud Run (Phase 2)

**What it is.** A managed platform that runs containers on request. It scales from
zero to N instances, you pay for usage, and there are no servers to patch. A
**service** has immutable **revisions**. Traffic can be split between revisions,
which makes rollback a single command.

**The container contract (what my images must do)**
- Listen on `$PORT` (8080 by default) on the **ingress container**.
- Be stateless: the local file system is in-memory and lost when the instance stops.
- Start quickly and handle `SIGTERM` (the PID 1 lessons from the original project
  apply directly).
- Log to stdout/stderr. Cloud Logging collects it automatically.

### Sidecars: keeping "one process per container"

Cloud Run supports **multiple containers per instance**. Exactly one is the ingress
container, the others are sidecars. They share `localhost` and can share in-memory
volumes. That lets the original NGINX → PHP-FPM split survive unchanged:

```
Cloud Run instance
├── nginx      (ingress, :8080)  ── fastcgi_pass 127.0.0.1:9000
└── wordpress  (sidecar, php-fpm :9000)
     container dependency: nginx starts after wordpress is ready (startup probe)
```

### Handling WordPress state

| What | Where it lives | Why |
|---|---|---|
| WordPress core, themes, plugins | **Baked into the image** | Immutable. Updates go through CI, not the admin panel (`DISALLOW_FILE_MODS`) |
| `wp-content/uploads` | Cloud Storage, through a Cloud Run **Cloud Storage volume mount** (FUSE) | Shared between instances, durable |
| Database | Cloud SQL | Managed |
| Secrets | Secret Manager volume mounts | Files, as in the original project |

The FUSE mount is not a POSIX disk (no file locking, higher latency). That is
acceptable for media uploads and is the reason code doesn't live there. If it
causes problems, the fallback is an offload plugin that writes to Cloud Storage
through the API.

**Planned settings:** `min-instances=0` (dev, cost) / `1` (demo, no cold start),
`max-instances=3` (cost ceiling), CPU allocated only during requests, startup CPU
boost on, ingress `internal-and-cloud-load-balancing`.

**References**
- Cloud Run overview: <https://cloud.google.com/run/docs/overview/what-is-cloud-run>
- Container runtime contract: <https://cloud.google.com/run/docs/container-contract>
- Multi-container (sidecars): <https://cloud.google.com/run/docs/deploying#sidecars>
- Cloud Storage volume mounts: <https://cloud.google.com/run/docs/configuring/services/cloud-storage-volume-mounts>
- Ingress settings: <https://cloud.google.com/run/docs/securing/ingress>
- Rollbacks and traffic migration: <https://cloud.google.com/run/docs/rollouts-rollbacks-traffic-migration>
- Connect to Cloud SQL from Cloud Run: <https://cloud.google.com/sql/docs/mysql/connect-run>

---

## GKE Autopilot (Phase 4, stretch)

**What it is.** Managed Kubernetes where Google also manages the nodes, and you pay per
pod resource request. The original roadmap already listed "port to Kubernetes (Helm
chart)". This is where that happens, with the same images.

**Why it isn't the main target.** For one small web application, Kubernetes adds
cost and operational complexity that Cloud Run doesn't. Explaining *why not* is a
valuable interview answer in itself.

**References**
- Autopilot overview: <https://cloud.google.com/kubernetes-engine/docs/concepts/autopilot-overview>

---

## Comparison to fill in after Phases 1 and 2

This table is one of the project's deliverables. It is filled in with measured
numbers, not guesses:

| Metric | Original (local VM) | Phase 1 (GCE) | Phase 2 (Cloud Run) |
|---|---|---|---|
| Monthly cost (idle / light traffic) | €0 (my laptop) | | |
| Time from clone to working site | | | |
| Time to deploy a code change | | | |
| Time to roll back | | | |
| Things I have to patch | Host OS, all images | Host OS, all images | Images only |
| Cold-start latency | — | — | |
| What happens when the host dies | Everything is down | Everything is down until recreated | Nothing (regional service) |
