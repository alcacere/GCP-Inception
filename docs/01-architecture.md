# 01 · Architecture and Design

## Design goals

1. **Reproducible from an empty project.** `make bootstrap && make apply` builds the
   whole platform. Nothing is created by clicking in the console. If something has to
   be created by hand (billing, domain purchase), it is written down in the
   [runbook](12-operations-runbook.md).
2. **Built, not pulled.** Application images come from my own Dockerfiles, are
   scanned in CI, and are deployed **by digest**, never by a mutable tag.
3. **One process per container.** It still applies on Cloud Run, through sidecars.
4. **A single, encrypted entry point.** Only the load balancer has a public IP. Cloud
   Run refuses direct traffic, Cloud SQL has no public IP, and SSH goes through IAP.
5. **No long-lived credentials anywhere.** There are no service account keys. CI uses
   Workload Identity Federation, workloads use their attached service account, and
   application passwords live in Secret Manager.
6. **Least privilege by default.** Every workload has its own service account with
   only the roles it needs.
7. **Observable from the user's point of view.** Uptime checks probe the public
   endpoint from several regions. Alerts are tied to an SLO, not just to "is it up".
8. **Cost is a design constraint.** Every resource has a known cost, expensive
   resources are optional, and `make destroy` removes everything.

## Target architecture (Phase 2)

```
                         Internet
                            │  HTTPS :443 only (no :80 forwarding rule)
                            ▼
              ┌─────────────────────────────────────────────────────────┐
              │ Global external Application Load Balancer               │
              │  • Google-managed certificate                           │
              │  • SSL policy: MODERN, min TLS 1.2                      │
              │  • Cloud Armor (WAF rules, rate limits)                 │
              │  • URL map                                              │
              └──────┬───────────────────────┬──────────────┬─────────┘
           /*        │                       │ /portfolio/* │ /grafana/*
                     ▼                       ▼              ▼
         Serverless NEG                 Backend bucket   IAP (my Google identity only)
                     │                  + Cloud CDN         │
                     │                                      ▼
                     │                   Cloud Run "grafana" (sa-grafana)
                     │                   dashboards + data sources provisioned as code
                     │                   ──reads──► Cloud Monitoring / Cloud Logging APIs
                     ▼
   ┌────────── Cloud Run service "wordpress" ─────────────┐
   │ ingress: internal-and-cloud-load-balancing           │
   │ ┌──────────────┐  FastCGI    ┌────────────────────┐  │
   │ │ nginx :8080  │─127.0.0.1──►│ php-fpm :9000      │  │
   │ │ (ingress     │   :9000     │ (WordPress, WP-CLI)│  │
   │ │  container)  │             └───┬────────┬───────┘  │
   │ └──────────────┘                 │        │          │
   │  secrets mounted as files ◄── Secret Manager         │
   │  service account: sa-wordpress                       │
   └──────────────────────────────────┼────────┼──────────┘
                   Direct VPC egress  │        │  Cloud Storage FUSE
                                      ▼        ▼  (wp-content/uploads)
 ┌──────────── VPC "inception" (custom mode) ───────────┐
 │  subnet run-egress 10.10.0.0/24                      │
 │  private service access range 10.20.0.0/16           │
 │     ├── Cloud SQL for MySQL (private IP only)        │
 │     └── Memorystore (optional, off by default)       │
 └──────────────────────────────────────────────────────┘

 Cloud Monitoring uptime checks ──HTTPS──► the load balancer (from several regions)
 Cloud Logging ◄── every request log, container stdout/stderr, audit logs
```

### Phase 1 architecture (rehost, for comparison)

```
Internet ──HTTPS :443──► static external IP ──► GCE VM (custom Packer image)
                                                 ├── Docker Compose: the original stack
                                                 ├── secrets: Secret Manager → tmpfs files
                                                 ├── /data on a persistent disk (snapshots)
                                                 │   (blackbox + prometheus replaced by grafana)
                                                 └── Ops Agent → Logging/Monitoring
Admin ──IAP TCP forwarding──► VM :22 (no public SSH), grafana :3000 (no public port)
```

Phase 1 is deliberately close to the original. Comparing the two phases (cost,
operational effort, failure modes) is one of the main things this project produces.
See the comparison table in [08](08-compute-and-containers.md).

## Request flow (Phase 2)

### A dynamic WordPress page

```
Browser
  │ 1. DNS (Cloud DNS): ${DOMAIN} → load balancer anycast IP
  │ 2. TLS handshake at the Google Front End closest to the user
  │    (Google-managed cert; SSL policy refuses < TLS 1.2)
  ▼
Cloud Armor evaluates rules (OWASP CRS, rate limit on /wp-login.php)
  ▼
URL map: path not under /portfolio/ → backend service → serverless NEG
  ▼
Cloud Run (scales from 0..N instances)
  │ 3. nginx container: try_files → /index.php → fastcgi_pass 127.0.0.1:9000
  │ 4. php-fpm sidecar runs WordPress
  │ 5. SQL over private IP (Direct VPC egress) → Cloud SQL
  │ 6. Uploaded media read through the Cloud Storage volume mount
  ▼
HTML → nginx → load balancer → browser
```

### A portfolio page

The URL map sends `/portfolio/*` to a **backend bucket**. Cloud CDN caches it at the
edge, so no container runs and the request costs almost nothing. This is the cloud
version of the original "static traffic is cheap" design.

## Planned repository layout

```
inception-gcp/
├── Makefile                     # Single entry point (same contract as the original)
├── PLAN.md                      # Phases and definitions of done
├── infra/                       # Replaces vm/: everything that creates cloud resources
│   ├── bootstrap/               # Run once with local state: APIs, state bucket, WIF, deployer SA
│   ├── modules/                 # Reusable building blocks, one concern each
│   │   ├── network/             # VPC, subnets, firewall, NAT, private service access
│   │   ├── iam/                 # Workload service accounts and role bindings
│   │   ├── secrets/             # Secret Manager secrets and their accessors
│   │   ├── database/            # Cloud SQL instance, database, user, backups
│   │   ├── cache/               # Memorystore (count = var.enable_cache ? 1 : 0)
│   │   ├── run-service/         # Cloud Run service with sidecars
│   │   ├── static-site/         # Bucket + backend bucket + CDN
│   │   ├── edge/                # LB, certificate, SSL policy, Cloud Armor, DNS
│   │   ├── vm/                  # Phase 1 Compute Engine host
│   │   └── observability/       # Uptime checks, alert policies, SLOs, Grafana service + IAP
│   └── envs/
│       ├── dev/                 # Calls the modules with dev variables + backend config
│       └── prod/                # Same modules, different variables (Phase 3)
├── packer/                      # Phase 1: googlecompute image (reuses the original provisioners)
├── srcs/
│   └── requirements/            # Same build-context-per-service layout as the original
│       ├── nginx/               # Now a Cloud Run ingress container (no TLS: the LB terminates it)
│       ├── wordpress/           # PHP-FPM, immutable image with plugins baked in
│       ├── website/             # Static files, synced to the bucket instead of served by darkhttpd
│       ├── grafana/             # Grafana image + provisioning/ (data sources, dashboards as JSON)
│       └── tools/               # Helper scripts (secret seeding, migration, smoke tests)
├── .github/workflows/           # ci.yml (pull requests), deploy.yml (main)
└── docs/                        # This documentation and the ADRs
```

### Why this structure

- **`infra/bootstrap` is separate** because it creates the state bucket that every
  other Terraform configuration stores its state in (the chicken-and-egg problem).
  It runs once, with local state that is then migrated into the bucket.
- **Modules vs environments.** Modules say *what* a component is. `envs/<env>` says
  *how big, where, and how many*. Adding `prod` means adding variables, not copying code.
- **`srcs/requirements` keeps the original layout**, so anyone who read the original
  project knows where to look, and the diff between the two projects is easy to show.
- **No `secrets/` directory.** Secret values never touch the repository or the
  workstation disk. See [05](05-identity-and-security.md).

## Ports and exposure (Phase 2)

| Component | Reachable from | Port | Notes |
|---|---|---|---|
| Load balancer | Internet | 443 | The only public endpoint. No port 80 forwarding rule, so plain HTTP is refused |
| Cloud Run `wordpress` | Load balancer only | 8080 (nginx) | The `*.run.app` URL returns 404 because of the ingress setting |
| PHP-FPM sidecar | nginx in the same instance | 9000 (localhost) | Not reachable outside the instance |
| Cloud SQL | VPC (private service access) | 3306 | No public IP. TLS required |
| Memorystore (optional) | VPC | 6379 | AUTH + in-transit encryption |
| Portfolio bucket | Load balancer (backend bucket) | — | Public read is needed for a backend bucket. Contains only public static files |
| Cloud Run `grafana` | Load balancer, **through IAP** | 3000 | Only identities with `iap.httpsResourceAccessor` get in. The `*.run.app` URL is closed |
