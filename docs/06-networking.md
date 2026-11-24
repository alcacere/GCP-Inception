# 06 · Networking

The original stack had one Docker bridge network and two published ports. On GCP the
equivalent is a **VPC** that the data tier lives in, with no public addresses, plus
a load balancer that is the only way in.

```
VPC "inception" (custom mode, global)
├── subnet run-egress   10.10.0.0/24  europe-west1   ← Cloud Run Direct VPC egress, Phase 1 VM
├── private service access range 10.20.0.0/16        ← peered to Google's producer network
│     ├── Cloud SQL (private IP)
│     └── Memorystore (optional)
├── Cloud Router + Cloud NAT                          ← outbound internet for private resources
└── firewall: default deny ingress; allow LB health checks and IAP only
```

---

## VPC networks and subnets

**What it is.** A GCP VPC is **global**. Its **subnets are regional**. In *auto mode*
a subnet is created in every region automatically. In **custom mode** you choose
exactly which subnets exist and their CIDR ranges.

**Why custom mode.** Predictable addressing, no unused subnets in other regions, and
it is what production networks use. The `default` network, which has permissive
firewall rules, is deleted in bootstrap.

**Planned addressing**

| Range | Purpose |
|---|---|
| `10.10.0.0/24` | `run-egress` subnet (Cloud Run instances take IPs from it with Direct VPC egress, and the Phase 1 VM) |
| `10.20.0.0/16` | Allocated range for private service access (Cloud SQL, Memorystore) |

**References**
- VPC overview: <https://cloud.google.com/vpc/docs/vpc>
- Subnets: <https://cloud.google.com/vpc/docs/subnets>

---

## Firewall rules

**What it is.** VPC firewall rules are **stateful**. They are applied per instance and
select targets by **network tag** or **service account**. Every VPC has an implied
*deny all ingress* and *allow all egress*.

**Planned rules**

| Rule | Direction | Source | Target | Ports | Phase |
|---|---|---|---|---|---|
| `allow-https` | ingress | `0.0.0.0/0` | SA `sa-host` | 443 | 1 |
| `allow-iap-ssh` | ingress | `35.235.240.0/20` (IAP) | SA `sa-host` | 22 | 1 |
| `allow-lb-health` | ingress | `130.211.0.0/22`, `35.191.0.0/16` | backends | serving port | if instance backends are used |
| *(implied)* | ingress | any | any | any: **deny** | all |

Rules target **service accounts rather than network tags**, because anyone who can
edit an instance can change its tags, but changing its service account needs IAM
permission.

Firewall rule logging is turned on for the deny rules during debugging.

**References**
- VPC firewall rules: <https://cloud.google.com/firewall/docs/firewalls>
- IAP source range: <https://cloud.google.com/iap/docs/using-tcp-forwarding#create-firewall-rule>
- Health check source ranges: <https://cloud.google.com/load-balancing/docs/health-check-concepts#ip-ranges>

---

## Cloud NAT

**What it is.** Lets resources **without external IPs** open outbound connections
(package downloads, external APIs) without accepting inbound ones. It is a managed,
software-defined service configured on a Cloud Router, not a VM.

**Why here.** In Phase 1 the VM has an external IP for serving traffic, but the goal
in later iterations is to put the VM behind a load balancer with **no external IP**.
NAT is what keeps `apt` and image pulls working then.

**References**
- Cloud NAT overview: <https://cloud.google.com/nat/docs/overview>

---

## Private service access (Cloud SQL and Memorystore)

**What it is.** Managed services such as Cloud SQL run in a Google-owned **producer
VPC**. Private service access reserves a range in *my* VPC and creates a **VPC
peering** to the producer network, so the instance gets a private IP from my range.

```
my VPC (10.20.0.0/16 allocated) ◄──VPC peering (servicenetworking)──► Google producer VPC
                                                                       └── Cloud SQL 10.20.0.3
```

**How this project will implement it.** `google_compute_global_address` (purpose
`VPC_PEERING`) + `google_service_networking_connection`, then the Cloud SQL instance
with `ipv4_enabled = false` and `private_network = <vpc>`.

**Newer alternative to know about.** Private Service Connect (PSC) for Cloud SQL
avoids peering (and its non-transitive routing limits). It is mentioned in the
roadmap. Private service access is used first because it is still the most common
pattern.

**References**
- Private services access: <https://cloud.google.com/vpc/docs/private-services-access>
- Cloud SQL private IP: <https://cloud.google.com/sql/docs/mysql/private-ip>
- Private Service Connect for Cloud SQL: <https://cloud.google.com/sql/docs/mysql/about-private-service-connect>

---

## Cloud Run → VPC: Direct VPC egress

**What it is.** Cloud Run instances don't live in my VPC by default. **Direct VPC
egress** gives each instance an IP in a chosen subnet, so it can reach private IPs
such as Cloud SQL. It replaces the older *Serverless VPC Access connector*, which ran
on VMs you paid for.

**Setting used:** `egress = PRIVATE_RANGES_ONLY`. Traffic to private IPs goes through
the VPC, and everything else uses Cloud Run's normal internet path.

**References**
- Direct VPC egress: <https://cloud.google.com/run/docs/configuring/vpc-direct-vpc>
- Compare with connectors: <https://cloud.google.com/run/docs/configuring/connecting-vpc>

---

## Cloud DNS

**What it is.** Managed authoritative DNS. A **public zone** for the domain holds the
`A` record pointing at the load balancer IP. The domain registrar's NS records are
set to the zone's name servers.

**Migration detail.** The TTL is lowered 24 hours before the cut-over from Phase 1 to
Phase 2, so the switch takes minutes instead of hours.

**References**
- Cloud DNS overview: <https://cloud.google.com/dns/docs/overview>
