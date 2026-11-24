# 07 · Edge: Load Balancing, TLS and WAF

In the original project, NGINX did four jobs: terminate TLS, enforce the TLS policy,
route paths to backends, and be the only open port. On GCP those jobs move to the
**Global external Application Load Balancer**, which runs on Google's edge network.
NGINX stays only for the FastCGI step ([ADR 0004](adr/0004-nginx-as-cloud-run-sidecar.md)).

| Job | Original | Phase 2 |
|---|---|---|
| Single public entry point | NGINX `:443` | One global anycast IP |
| TLS termination | NGINX + private CA | Google Front End + Google-managed certificate |
| TLS policy | `ssl_protocols TLSv1.2 TLSv1.3` | SSL policy, profile `MODERN`, min TLS 1.2 |
| Path routing | `location` blocks | URL map |
| Static files | NGINX reads the volume | Backend bucket + Cloud CDN |
| Monitoring UI | `/prometheus/` proxied by NGINX | `/grafana/` backend service with IAP |
| Abuse protection | — | Cloud Armor |

---

## Global external Application Load Balancer

**What it is.** A layer 7 HTTP(S) proxy load balancer made of several resources that
are easy to confuse. Learning the chain is a classic certification topic:

```
global address (static anycast IP)
  └── forwarding rule :443
        └── target HTTPS proxy ── certificate(s) + SSL policy
              └── URL map  (host/path rules)
                    ├── backend service ── serverless NEG ── Cloud Run "wordpress"
                    │      └── Cloud Armor policy, logging, (IAP optional)
                    ├── backend bucket ── gs://…-portfolio  (Cloud CDN on)
                    └── backend service ── serverless NEG ── Cloud Run "grafana"
                           └── IAP on: only allowed Google identities reach it
```

- **Serverless NEG** (network endpoint group): how a load balancer points at a Cloud
  Run service. Cloud Run manages health on its own, so no health check is configured.
- **No port 80.** No forwarding rule is created for `:80`, so plaintext HTTP is refused
  at the TCP level, exactly like the original project. Adding an HTTP→HTTPS redirect
  would need a second forwarding rule (an extra cost) and is listed as optional.

**References**
- Application Load Balancer overview: <https://cloud.google.com/load-balancing/docs/https>
- Serverless NEGs: <https://cloud.google.com/load-balancing/docs/negs/serverless-neg-concepts>
- Set up an LB with Cloud Run: <https://cloud.google.com/load-balancing/docs/https/setup-global-ext-https-serverless>
- URL maps: <https://cloud.google.com/load-balancing/docs/url-map-concepts>

---

## Certificates

**What it is.** **Google-managed certificates** are issued and renewed automatically
for domains that point at the load balancer. Provisioning only completes once DNS
resolves to the LB IP, which is a common source of "stuck in PROVISIONING" confusion.
**Certificate Manager** is the newer API (certificate maps, DNS authorization, which
lets the certificate be issued *before* the DNS cut-over).

**How this project will implement it.** Certificate Manager with **DNS
authorization**. The certificate is ready before traffic moves from Phase 1, so the
switch has no TLS gap.

**What happened to the private CA?** It was right for a closed lab: it taught
X.509, chains and name constraints. A public website needs a certificate browsers
already trust, so the CA is retired here. The original docs still explain it.

**References**
- Google-managed SSL certificates: <https://cloud.google.com/load-balancing/docs/ssl-certificates/google-managed-certs>
- Certificate Manager overview: <https://cloud.google.com/certificate-manager/docs/overview>
- DNS authorization: <https://cloud.google.com/certificate-manager/docs/dns-authorizations>

---

## SSL policies

**What it is.** Controls the TLS versions and cipher suites the load balancer
negotiates. Profiles: `COMPATIBLE` (widest), `MODERN`, `RESTRICTED` (strictest), or
`CUSTOM`. A **minimum TLS version** is set separately.

**Setting:** `MODERN` + `min_tls_version = TLS_1_2`. This is the same guarantee the
original NGINX config made, and it is verified the same way:

```sh
openssl s_client -connect ${DOMAIN}:443 -tls1_1 </dev/null   # must fail
openssl s_client -connect ${DOMAIN}:443 -tls1_2 </dev/null   # must succeed
```

**References**
- SSL policies: <https://cloud.google.com/load-balancing/docs/ssl-policies-concepts>

---

## Cloud Armor

**What it is.** A web application firewall and DDoS protection service attached to
backend services. A **security policy** is an ordered list of rules (allow, deny,
throttle, rate-based ban). It includes **preconfigured WAF rules** based on the OWASP
ModSecurity Core Rule Set (SQL injection, XSS, LFI, and others).

**Planned policy**

| Priority | Rule | Action |
|---|---|---|
| 1000 | `evaluatePreconfiguredWaf('sqli-v33-stable')` | deny(403) |
| 1010 | `evaluatePreconfiguredWaf('xss-v33-stable')` | deny(403) |
| 2000 | path `/wp-login.php` or `/xmlrpc.php`, more than N requests/min per IP | rate-based ban |
| 2010 | path `/xmlrpc.php` | deny(403) (not used by this site) |
| default | everything else | allow |

New rules are first deployed in **preview mode**, which logs what *would* be blocked
without blocking it, and are enforced once the logs show no false positives. This is
standard practice for WAF changes.

**References**
- Cloud Armor overview: <https://cloud.google.com/armor/docs/cloud-armor-overview>
- Preconfigured WAF rules: <https://cloud.google.com/armor/docs/waf-rules>
- Rate limiting: <https://cloud.google.com/armor/docs/rate-limiting-overview>

---

## Cloud CDN and the portfolio bucket

**What it is.** Cloud CDN caches responses at Google's edge locations. Turned on for a
**backend bucket**, it serves the static portfolio (and the snake game) without
compute, with cache control from the object's `Cache-Control` metadata.

**Security note.** A backend bucket needs public read access (`allUsers` with
`roles/storage.objectViewer`). That is acceptable *only* because this bucket holds
nothing but public static files. Every other bucket keeps public access prevention
enforced.

**References**
- Cloud CDN overview: <https://cloud.google.com/cdn/docs/overview>
- Backend buckets: <https://cloud.google.com/load-balancing/docs/backend-bucket>
- Host a static website: <https://cloud.google.com/storage/docs/hosting-static-website>
