# 11 — Limitations and possible improvements

An honest list of what the platform does **not** do yet, why, and what we would do next.

## Known limitations

| # | Limitation | Impact | Mitigation today |
|---|---|---|---|
| 1 | **One shared database** (MongoDB Atlas M0, single region in AWS Mumbai) | the database is a shared dependency for all three clouds; the problem statement's "replicate databases across clouds" is met by encrypted copies, not live replication | nightly encrypted backups in GCS, S3 and Azure Blob + restore drills; RPO ≤ 24 h |
| 2 | **Terraform is applied by Ansible from the admin laptop**, not by a GitHub Actions pipeline | the problem statement asks for CI/CD of Terraform; changes are reviewed in pull requests but applied by hand | every apply is guarded (plan → check change types → apply that plan); the GitHub plan/apply identities already exist in each `bootstrap/` |
| 3 | **Jenkins and the central Grafana run in GKE** (the primary) | during a GCP outage there is no automatic deploy and no central dashboard | break-glass `deploy.yml`; each cloud's Alertmanager still alerts Slack |
| 4 | Backups run only while the platform is up (Jenkins cron) | no backups between sessions (nothing is written then either) | backup-on-demand job; take one before `down.yml` if data changed |
| 5 | `notification-service` keeps Socket.IO chat state in memory | runs as 1 replica per cloud; a restart drops open chat sessions | — |
| 6 | Waking a standby takes minutes (RTO 3 min 42 s, Azure ≈ 6 min) | a GCP outage is visible to shoppers for a few minutes | deliberate cost trade-off (−60 % node cost) |
| 7 | Failover Worker uses a fine-grained **personal access token** | a long-lived secret (scoped to one repo, `actions:write`) | stored only in GCP Secret Manager and as a Worker secret |
| 8 | No Cloudflare WAF custom rules / rate limiting | only Cloudflare's built-in DDoS protection | origins accept only Cloudflare IPs |
| 9 | Standby sleep issues found in the final test | AWS sleep can take 30 min (PDB drain); stopping an AKS that is still starting fails | manual commands in the runbook |
| 10 | Teardown can leave an AWS NLB behind | `down.yml` stops safely; manual cleanup needed | leftover check + runbook §7 |
| 11 | Zero downtime proven on 29 Oct 2026, not in the final test | the final-test probe stopped before the deploy started | repeat the CI/CD test with the probe running for the whole deploy |

## Possible improvements

| Priority | Item | Why |
|---|---|---|
| 1 | Fix standby sleep: drain AWS nodes with `--disable-eviction` and wait until instances are terminated; wait for an in-progress AKS start before stopping | correct "asleep" state, no hidden cost |
| 2 | Order the AWS add-on teardown so the LB Controller removes Envoy's NLB before it is uninstalled | `down.yml` without manual cleanup |
| 3 | Preflight checks for the Atlas API key access list, the current IP and GitHub download reachability; correct per-cloud task labels | catch problems in seconds instead of mid-run |
| 4 | Terraform plan on every pull request and apply after approval via GitHub Actions (identities exist) | full GitOps for infrastructure |
| 5 | Replace the PAT with a GitHub App installation token | no long-lived personal token |
| 6 | Atlas on a dedicated tier with multi-region / multi-cloud nodes | live database replication across clouds |
| 7 | Move Jenkins (or a second runner) and Grafana out of the primary cloud | CD and central monitoring survive a GCP outage |
| 8 | Redis adapter for Socket.IO | notification-service can scale beyond 1 replica |
| 9 | Cloudflare WAF managed rules and rate limiting (paid plan) | application-layer protection |
