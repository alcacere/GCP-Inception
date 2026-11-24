# 12 · Operations Runbook

> Planned procedures. Each command is checked and corrected while its phase is
> implemented. `${PROJECT}`, `${REGION}` and `${DOMAIN}` come from `infra/envs/<env>/terraform.tfvars`.

## Manual prerequisites (the only steps outside code)

1. A Google account with a Cloud Billing account (free trial if available).
2. A domain name, with its NS records pointed at Cloud DNS after the first apply.
3. Local tools: `gcloud`, `terraform` (or `tofu`), `packer` (Phase 1), `docker`, `make`.
4. `gcloud auth login` and `gcloud auth application-default login`.

## Deploy from a fresh clone

```sh
make bootstrap ENV=dev     # APIs, state bucket, WIF, deployer SA, Artifact Registry
make secrets   ENV=dev     # generate values locally → Secret Manager (never written to disk)
make image               # Phase 1 only: Packer builds family/inception-host
make apply     ENV=dev     # everything else
make push && make deploy   # Phase 2: images by digest → Cloud Run
make smoke                 # verification checks below
```

## Verification checklist

### Edge and TLS
```sh
curl -sI https://${DOMAIN}/ | head -1                              # HTTP/2 200
openssl s_client -connect ${DOMAIN}:443 -tls1_1 </dev/null          # must FAIL
openssl s_client -connect ${DOMAIN}:443 -tls1_2 </dev/null | grep -E 'Protocol|Verify return code: 0'
curl -s --max-time 5 http://${DOMAIN}/ ; echo "exit=$?"             # must fail: no :80
gcloud compute ssl-policies describe inception-ssl --format='value(minTlsVersion,profile)'
```

### Nothing is reachable except the load balancer
```sh
# The Cloud Run URL refuses direct traffic (ingress = internal-and-cloud-load-balancing)
curl -s -o /dev/null -w '%{http_code}\n' "$(gcloud run services describe wordpress \
  --region ${REGION} --format='value(status.url)')"                 # 404 expected

# Cloud SQL has no public IP
gcloud sql instances describe inception-dev-sql \
  --format='value(settings.ipConfiguration.ipv4Enabled)'           # False

# Phase 1: no public SSH, the database port is closed from outside
nc -vz -w 3 ${VM_IP} 22 ; nc -vz -w 3 ${VM_IP} 3306                # both must fail
```

### Identity and secrets
```sh
gcloud iam service-accounts keys list --iam-account sa-deployer@${PROJECT}.iam.gserviceaccount.com \
  --managed-by=user                                                 # empty
gcloud run services describe wordpress --region ${REGION} --format=yaml \
  | grep -iE 'password|secret'          # only secret *references*, never values
terraform -chdir=infra/envs/dev show -json | grep -c '"secret_data"' # 0
```

### Grafana is identity-gated
```sh
# Without a Google identity, IAP answers instead of Grafana
curl -s -o /dev/null -w '%{http_code} %{redirect_url}\n' https://${DOMAIN}/grafana/   # 302 → accounts.google.com
# Grafana's own login page is never reachable directly
curl -s -o /dev/null -w '%{http_code}\n' "$(gcloud run services describe grafana \
  --region ${REGION} --format='value(status.url)')"                 # 404 expected
# Dashboards survive a redeploy (they are provisioned from git)
gcloud run services update grafana --region ${REGION} --update-labels=redeploy=$(date +%s)
```

### WAF
```sh
curl -s -o /dev/null -w '%{http_code}\n' "https://${DOMAIN}/?id=1%27%20OR%201=1--"  # 403
```

### Resilience
```sh
# Rollback: send traffic back to the previous revision
gcloud run services update-traffic wordpress --region ${REGION} --to-revisions=<prev>=100

# Data survives a full redeploy of the compute layer
terraform -chdir=infra/envs/dev apply -replace='module.wordpress.google_cloud_run_v2_service.this'
```

## Day-2 operations

| Task | Command / procedure |
|---|---|
| Follow logs | `gcloud beta run services logs tail wordpress --region ${REGION}` |
| Open Grafana (Phase 1) | `gcloud compute start-iap-tunnel inception-host 3000 --local-host-port=localhost:3000 --zone ${ZONE}` → <http://localhost:3000> |
| Open Grafana (Phase 2) | `https://${DOMAIN}/grafana/` (IAP sign-in) |
| Change a dashboard | Edit in the UI → *Export → JSON* → commit to `srcs/requirements/grafana/provisioning/dashboards/` → PR → deploy |
| SSH to the Phase 1 host | `gcloud compute ssh inception-host --zone ${ZONE} --tunnel-through-iap` |
| Database shell | Cloud SQL Studio (console), or `gcloud sql connect` through the Auth Proxy |
| Rotate a secret | `make secrets ROTATE=db-password` → new version → update the DB user → new Cloud Run revision → disable the old version |
| Restore the database to a point in time | `gcloud sql instances clone inception-dev-sql inception-dev-sql-restore --point-in-time=<RFC3339>` |
| Restore an upload | `gcloud storage ls -a gs://…-uploads/<path>` → copy the wanted generation back |
| Scale to zero for the night | `min-instances=0` is the default. In Phase 1: `gcloud compute instances stop` |

## Teardown (cost safety net)

```sh
make destroy ENV=dev     # terraform destroy for envs/dev
# Bootstrap resources (state bucket, WIF) stay on purpose. Remove them with:
make destroy-bootstrap   # asks for confirmation
# Or, the nuclear option: shut the whole project down (30-day recovery window)
gcloud projects delete ${PROJECT}
```

Then check the **Billing → Reports** page the next day, filtered by the `project=inception` label.

## Troubleshooting

| Symptom | Likely cause | Check |
|---|---|---|
| Certificate stuck in `PROVISIONING` | DNS doesn't point at the LB yet, or the DNS authorization record is missing | `dig +short ${DOMAIN}`, `gcloud certificate-manager certificates describe` |
| 502 from the LB | Cloud Run revision failed to start | Revision logs, the startup probe, `PORT` mismatch |
| WordPress "Error establishing a database connection" | VPC egress not configured, wrong private IP, missing `cloudsql.client`, wrong password version | Revision env/secret config, Cloud SQL logs |
| `PERMISSION_DENIED` in CI | WIF attribute condition doesn't match the repo/ref, or a missing `iam.serviceAccountUser` | STS error message in the job log, the `principalSet` in the SA's IAM policy |
| `terraform apply` hangs on lock | A previous run was interrupted | `terraform force-unlock <id>` (only after confirming nothing is running) |
| Uploads disappear | Written outside the mounted path | Check `wp-content/uploads` is the mount point |
| Grafana panels say "permission denied" | `sa-grafana` is missing `monitoring.viewer` / `logging.viewer`, or the data source uses the wrong project | Grafana server log, the SA's IAM bindings |
| Grafana redirects in a loop under `/grafana/` | `root_url` / `serve_from_sub_path` don't match the URL map path | `GF_SERVER_ROOT_URL` in the revision |
| Unexpected charges | A forgotten resource or toggle | Billing report grouped by SKU and label |
