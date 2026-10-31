# 09 — Final end-to-end test report

**When:** 30 October 2026 14:55 UTC → 31 October 2026 (teardown)
**Who:** Rishav Mukhopadhyay (operator) — commands from WSL Ubuntu / PowerShell, evidence by screenshot
**Scope:** everything at once — start from code, run the shop in three clouds, release through CI/CD, monitor, sleep the standbys, back up and restore, fail over, tear down.

## Summary

| Part | Test | Result |
|---|---|---|
| A | Permanent foundations: repos, CI, registries, backups, state, identities, budgets | ✅ all present |
| B | Session start with `up.yml` (3 clusters in parallel) | ✅ `failed=0` everywhere, 23 min 13 s ¹ |
| C | Shop, clusters, global layer and database in all three clouds | ✅ 6/6 services ready ×3, `/api/health` ok ×3 |
| D | One merge → CI → same image in 3 registries → Jenkins → 3 clouds | ✅ live in gcp, azure, aws in 4 min 57 s ² |
| E | Dashboards, logs, alert → Slack → resolved, cost | ✅ |
| F | Standbys to sleep; Jenkins skips them | ✅ (AWS sleep slow — issue 4) |
| G | Backup to 3 clouds · restore from 3 clouds · automatic failover | ✅ 3/3 restores · **RTO 3 min 42 s** |
| H | Teardown with `down.yml` | ✅ after manual fixes (issues 5–6) |

¹ GKE and AKS already existed from the first attempt (see issue 1); EKS was built from scratch. ² Zero downtime during this deploy was **not** measured — the probe was stopped before the deploy began (see §D).

### Issues found during the test

| # | Issue | Handled |
|---|---|---|
| 1 | AWS SSO session expired → AWS dropped out of the first `up.yml`, then the global layer failed | `aws sso login`, re-run — [Troubleshooting](10-troubleshooting.md#aws-steps-fail-with-refresh-cached-sso-token-failed--invalidrequestexception) |
| 2 | Operator's public IP changed twice (home ↔ phone hotspot) → cluster APIs and Atlas API refused | IP updated in tfvars, Atlas key, clusters re-applied — [Runbook §6](05-operations-runbook.md#6-your-public-ip-changed-mid-session) |
| 3 | Helm chart downloads timed out — one GitHub CDN address unreachable from the ISP | address pinned — [Troubleshooting](10-troubleshooting.md#wsl-downloads-time-out-after-30-s) |
| 4 | AWS standby sleep stuck ~30 min (PDB drain); Azure stop refused during an in-progress start | manual drain / wait — see [Limitations](11-limitations-and-improvements.md) |
| 5 | AWS NLB left behind → `down.yml` safety stop kept EKS | NLB, target group, security groups deleted by hand, re-run |
| 6 | Atlas IP access list contained a manual `0.0.0.0/0` | removed |

---

## A. Permanent foundations

These exist between sessions and cost (almost) nothing.

**GitHub organisation and branch protection**

![Repositories](images/a01-github-org-repositories.png)
![Branch protection](images/a02-branch-protection.png)
![Required checks](images/a02-required-checks.png)

**CI run, security scan and shared workflows**

![CI run graph](images/a03-ci-run-graph.png)
![Trivy steps](images/a05-trivy-scan-steps.png)
![Reusable workflows](images/a06-reusable-workflows.png)

**The same images in three registries**

![ECR](images/a08-ecr-repositories.png)
![ACR](images/a09-acr-repositories.png)
![Artifact Registry](images/a10-artifact-registry.png)

**Encrypted backups in three clouds**

![S3 backups](images/a11-s3-backups.png)
![Azure backup containers](images/a12-azure-backup-containers.png)
![GCS backups](images/a13-gcs-backups.png)

**Terraform state (versioned)**

![S3 state](images/a14-s3-terraform-state.png)
![Azure state](images/a15-azure-terraform-state.png)
![GCS state](images/a16-gcs-terraform-state.png)

**Keyless identities**

![AWS roles](images/a17-aws-iam-roles.png)
![AWS GitHub OIDC provider](images/a18-aws-github-oidc-provider.png)
![Azure managed identities](images/a19-azure-managed-identities.png)
![GCP workload identity pool](images/a20-gcp-workload-identity-pool.png)
![GCP service accounts](images/a21-gcp-service-accounts.png)
![GKE egress IP (permanent)](images/a22-gcp-prod-egress-ip.png)

**Budgets**

![AWS budget](images/a23-aws-budget.png)
![Azure budget](images/a24-azure-budget.png)
![GCP budget](images/a25-gcp-budget.png)

## B. Session start (`up.yml`)

`ansible-playbook playbooks/up.yml -e keep_awake=true` (standbys kept awake for the photos).

![Preflight](images/b01-preflight.png)
![PLAY RECAP — failed=0, 23 min 13 s](images/b04-up-recap-23m.png)

## C. Everything running

**Inside the clusters**

![GKE nodes and pods](images/c01-gke-nodes-pods.png)
![AKS nodes and pods](images/c02-aks-nodes-pods.png)
![EKS nodes and pods](images/c03-eks-nodes-pods.png)
![Deployments, HPA, PDB](images/c04-deploy-hpa-pdb.png)
![Secrets synced from the vault](images/c05-external-secrets-synced.png)
![Envoy Gateway and routes](images/c06-gateway-routes.png)
![Helm releases in GKE](images/c07-helm-releases-gke.png)
![/api/health in all three clouds](images/c08-health-all-clouds.png)

**The shop** — sign up, browse, cart, order; the same user and order are visible from every cloud (one shared database).

![Home](images/c09-shop-home.png)
![Valid TLS certificate](images/c10-tls-certificate.png)
![Catalogue](images/c11-shop-catalog.png)
![Sign up](images/c12-sign-up.png)
![Account](images/c12-account.png)
![Cart](images/c13-cart.png)
![Order complete](images/c13-order-complete.png)
![Order history](images/c13-order-history.png)
![Served by GCP](images/c15-shop-gcp.png)
![Served by Azure](images/c16-shop-azure.png)
![Served by AWS](images/c17-shop-aws.png)

**Cloud consoles**

![GKE clusters](images/c18-gke-clusters.png)
![GKE node pool](images/c19-gke-node-pool.png)
![GCE VMs](images/c20-gce-vms.png)
![Cloud NAT](images/c21-gcp-cloud-nat.png)
![Secret Manager](images/c22-gcp-secret-manager.png)
![AKS overview](images/c23-aks-overview.png)
![AKS node pool](images/c24-aks-node-pool.png)
![AKS workloads](images/c25-aks-workloads.png)
![AKS VM scale set](images/c26-aks-vmss-instances.png)
![Key Vault](images/c27-azure-key-vault.png)
![EKS overview](images/c28-eks-overview.png)
![EKS compute](images/c29-eks-compute.png)
![EC2 nodes](images/c30-ec2-nodes.png)
![AWS NLB](images/c31-aws-nlb.png)

**Global layer and database**

![Cloudflare DNS](images/c33-cloudflare-dns.png)
![Load balancer — 3 healthy pools](images/c34-cloudflare-lb-healthy.png)
![Traffic steering off = failover order](images/c35-cloudflare-steering-off.png)
![Monitors GET /api/health](images/c36-cloudflare-monitors.png)
![TLS Full (strict)](images/c37-cloudflare-tls-full-strict.png)
![Access applications](images/c38-cloudflare-access-apps.png)
![Failover Worker](images/c39-cloudflare-failover-worker.png)
![Atlas cluster](images/c40-atlas-cluster.png)
![Atlas database users](images/c41-atlas-db-users.png)
![Atlas IP access list (after removing 0.0.0.0/0)](images/c42-atlas-ip-access-list.png)
![Collections](images/c43-atlas-collections.png)

## D. CI → CD: one merge, three clouds

A one-line documentation change in `catalog-service` (PR #7) went through the whole pipeline.

![PR checks running](images/d02-pr-checks-running.png)
![PR merged](images/d03-pr-merged.png)
![main run green, including "Ask Jenkins to deploy"](images/d04-main-run-green.png)
![Image main-ba5cc42 pushed to ECR, ACR, Artifact Registry](images/d05-pushed-three-registries.png)
![Jenkins started GCP → Azure → AWS](images/d06-jenkins-triggered.png)
![Jenkins jobs](images/d07-jenkins-jobs.png)
![deploy-mern #1 — 4 min 57 s](images/d08-deploy-build.png)
![Helm rolling upgrade](images/d09-deploy-console-helm.png)
![Slack: live in gcp, azure, aws](images/d11-slack-deploy-live.png)
![Same tag running in all three clouds](images/d13-same-tag-three-clouds.png)
![Version recorded for the next session](images/d14-image-tags-file.png)

**Zero-downtime measurement.** The probe below asks each cloud for `/api/category/list` twice in parallel, continuously, and counts every answer that is not 200. In this test it ran from 18:09 to 18:10 UTC, but the deploy started at 18:19 UTC, so its "0 failed" result is **not** evidence for this deploy and is not shown. Zero downtime was demonstrated on 29 Oct 2026 (catalog-service PR #6, 0 failed requests). To repeat the measurement, start the probe **before** merging and stop it only after Slack reports *live in gcp,azure,aws*:

```bash
#!/bin/bash
# zdt.sh — zero-downtime probe. Ctrl+C prints the totals.
URL_PATH=/api/category/list; LOG=~/zdt-$(date +%H%M).csv; echo "time,cloud,code" > "$LOG"
worker() { while true; do code=$(curl -s -o /dev/null -w '%{http_code}' --max-time 10 "https://$1.rishavmukhopadhyay.store${URL_PATH}"); echo "$(date +%T),$1,${code}" >> "$LOG"; done; }
count() { grep -c ",$1," "$LOG"; }; failed() { grep ",$1," "$LOG" | grep -vc ',200$'; }
for c in gcp azure aws; do worker "$c" & worker "$c" & done
trap 'kill $(jobs -p) 2>/dev/null; for c in gcp azure aws; do echo "$c: $(count $c) requests, $(failed $c) failed"; done; exit' INT
while true; do sleep 10; l="$(date +%T)"; for c in gcp azure aws; do l="$l | $c $(count $c) req, $(failed $c) failed"; done; echo "$l"; done
```

## E. Monitoring, alerts and cost

**Dashboards and logs**

![Grafana data sources](images/e01-grafana-datasources.png)
![Overview — GCP](images/e02-overview-gcp.png)
![Overview — AWS](images/e03-overview-aws.png)
![Overview — Azure](images/e03-overview-azure.png)
![p95 latency](images/e04-p95-latency-gcp.png)
![Logs dashboard](images/e05-logs-dashboard.png)
![Loki explore](images/e06-loki-explore.png)

**Alert test:** catalog-service scaled to 0 in AWS → the AWS pool turned critical, Slack received the Cloudflare pool alert and **MernServiceDown — aws**; the shop stayed up (served by GCP). Scaled back → RESOLVED.

![Scale to zero](images/e07-scale-catalog-to-zero.png)
![AWS pool critical](images/e08-aws-pool-critical.png)
![Slack alerts](images/e11-slack-pool-and-servicedown.png)
![Restored](images/e12-catalog-restored.png)
![Resolved](images/e12-slack-resolved.png)
![Pools healthy again](images/e12-pools-healthy-again.png)

**Cost**

![OpenCost](images/e13-opencost-home.png)
![OpenCost by namespace](images/e13-opencost-by-namespace.png)
![Run-rate GCP $140.30](images/e14-runrate-gcp.png)
![Run-rate Azure $81.17](images/e14-runrate-azure.png)
![Run-rate AWS $130.82](images/e14-runrate-aws.png)
![AWS Cost Explorer — $6.76](images/e15-aws-cost-explorer.png)
![Azure cost analysis — ₹349.61](images/e16-azure-cost-analysis.png)
![GCP billing — covered by credits](images/e17-gcp-billing-report.png)

## F. Standbys asleep

`ansible-playbook playbooks/standby.yml`: AKS stopped, EKS node group at 0, GKE still serving.

![AKS stopped](images/f02-aks-stopped.png)
![AKS VM scale set empty](images/f03-aks-vmss-empty.png)
![EKS node group desired 0](images/f04-eks-nodegroup-zero.png)
![EC2 shutting down](images/f05-ec2-shutting-down.png)
![GKE still running](images/f06-gke-still-running.png)
![Standby pools down, GCP healthy](images/f07-pools-standbys-down.png)
![Slack: standby pools](images/f09-slack-standby-pools.png)

Jenkins `deploy-mern` with the running tag: GCP deployed, Azure and AWS skipped as sleeping standbys.

![Azure skipped](images/f11-jenkins-azure-skipped.png)
![AWS skipped](images/f11-jenkins-aws-skipped.png)
![Slack: asleep, skipped](images/f12-slack-deploy-skipped.png)

## G. Disaster recovery

### G1. Backup to three clouds

![backup-mongo](images/g01-backup-job.png)
![Build](images/g01-backup-build.png)
![Manifest with SHA-256 and document counts](images/g02-backup-manifest.png)
![Slack](images/g03-slack-backup.png)
![Newest backup in S3](images/g04-s3-newest-backup.png)
![Newest backup in GCS](images/g04-gcs-newest-backup.png)

### G2. Restore drill from three clouds

`ansible-playbook playbooks/drill-restore.yml`

![Restore drill PASSED](images/g05-restore-drill-passed.png)
![Slack: backup and three verified restores](images/g05-slack-backup-and-restores.png)
![restore-drill build](images/g06-restore-drill-build.png)

### G3. Failover drill: GCP goes down

`ansible-playbook playbooks/drill-failover.yml` — GCP's gateway scaled to 0 while both standbys slept; the wake-up was **automatic** (Cloudflare → Worker → GitHub Actions).

![All pools critical — the outage is real](images/g08-all-pools-critical.png)
![Slack: GCP pool down, MernServiceDown gcp](images/g13-slack-gcp-down.png)
![AKS waking](images/g10-aks-waking.png)
![failover-wake run started by the Worker](images/g11-failover-wake-run.png)
![Wake jobs: AWS 2 min 48 s, Azure ≈ 6 min](images/g11-failover-wake-jobs.png)
![Shop recovered](images/g12-shop-recovered-pools.png)
![Drill report — RTO 3 min 42 s](images/g14-failover-rto-report.png)

## H. Teardown

`ansible-playbook playbooks/down.yml` — wake standbys → add-ons destroy → state empty → leftover check → cluster destroy, AKS/EKS first, GKE last, then the global layer.

![down.yml preflight](images/h01-down-preflight.png)

The teardown needed four runs because of issues 1–3 and 5 above; each stopped safely with nothing half-deleted. Final state: see the end-of-session check in [Runbook §2](05-operations-runbook.md#2-end-a-session).
