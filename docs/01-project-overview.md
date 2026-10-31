# 01 — Project overview

## The problem

> *"In multi-cloud environments, managing resources across multiple cloud providers (AWS, Azure, GCP) can be complex and error-prone, especially when attempting to maintain consistent configurations and enable failover across clouds."* — Capstone Problem Statement #5

The task is a Terraform-based solution that deploys an application to **AWS, Azure and GCP**, keeps their configuration consistent, fails over automatically when a cloud breaks, recovers data after a disaster, and shows health and cost of everything in one place.

## What we built

One real application — a **MERN e-commerce shop** split into six microservices in three languages — running as three identical copies, one per cloud, behind one domain name.

| Building block | In one sentence |
|---|---|
| **Three clusters** | GKE (primary, always on), AKS and EKS (standbys that sleep like a stopped EC2 instance and wake in minutes). |
| **One front door** | Cloudflare sends every shopper to the first healthy cloud in the order GCP → Azure → AWS and checks each cloud's `/api/health` every 60 s. |
| **One database** | MongoDB Atlas; each cloud has its own database user and its own fixed outgoing IP on the allow-list. |
| **Everything as code** | Terraform for all infrastructure (13 stacks in 4 platform repos), Helm for the workloads, Ansible to run a session in one command. |
| **Keyless security** | No cloud key is stored anywhere. GitHub Actions, Jenkins and the pods prove who they are with short-lived OIDC / workload-identity tokens. |
| **CI → CD** | A merged pull request is tested, scanned, pushed to three registries and deployed cloud-by-cloud with automatic rollback. |
| **Observability** | Prometheus, Loki and OpenCost in each cluster; one central Grafana; alerts to Slack. |
| **Disaster recovery** | Nightly encrypted database backups in all three clouds, plus restore drills that prove they work. |

## Goals → how they are met → proof

| Project goal (problem statement) | How it is met | Proof |
|---|---|---|
| 1. Multi-cloud deployment with Terraform for high availability | Terraform stacks per cloud (`bootstrap`, `foundation`, `cluster`, `addons`) + `global-platform`; same Helm chart and same image in all clouds | [Architecture](02-architecture.md), [Test §C](09-test-report.md#c-everything-running) |
| 2. Automate infrastructure and configuration with reusable modules | Per-cloud `modules/` (network, cluster), a versioned observability module (`v1.1.0`) used by all three clusters, reusable GitHub workflows (`@v2`), Ansible playbooks with guarded `terraform apply` | [Setup guide](04-setup-guide.md), [Runbook](05-operations-runbook.md) |
| 3. Load balancing and failover across clouds | Cloudflare Load Balancer, 3 pools, `/api/health` monitor, failover order; stopped standbys woken automatically by Cloudflare → Worker → GitHub Actions | [Failover](08-disaster-recovery-and-failover.md), [Test §G](09-test-report.md#g3-failover-drill-gcp-goes-down) — RTO **3 min 42 s** |
| 4. Disaster recovery, replication across clouds, tested failover | `age`-encrypted `mongodump` stored in GCS, S3 and Azure Blob; restore drill restores each copy into a scratch DB and compares document counts | [Test §G](09-test-report.md#g1-backup-to-three-clouds) — **3/3 restores passed** |
| 5. Monitoring, alerts and cost visibility | kube-prometheus-stack + Loki + OpenCost per cloud, central Grafana, 5 custom alerts + Cloudflare pool alerts → Slack, cloud budgets | [Monitoring & cost](07-monitoring-and-cost.md), [Test §E](09-test-report.md#e-monitoring-alerts-and-cost) |

### Tools asked for vs. tools used

| Asked for | Used | Note |
|---|---|---|
| Terraform (HCL) | Terraform 1.16.6 | all infrastructure, including DNS, load balancer, database users and vault secrets |
| Python (optional) | Python | commerce service; Ansible is Python-based |
| AWS, Azure, GCP | all three | Mumbai / Central India regions |
| Vault for credentials | each cloud's own vault (Secret Manager, Key Vault, SSM Parameter Store) + External Secrets Operator | a cloud's secrets never leave that cloud; no static cloud keys exist at all (OIDC) |
| Prometheus + Grafana | kube-prometheus-stack in every cluster + central Grafana | plus Loki/Alloy for logs and OpenCost for cost |
| PagerDuty or Slack | Slack | Alertmanager, Cloudflare notifications, Jenkins |
| GitHub/GitLab CI/CD | GitHub Actions (CI, image publishing, failover wake-up) + Jenkins (CD) | Terraform itself is applied through guarded Ansible playbooks — see [Limitations](11-limitations-and-improvements.md) |

### Deliverables

| Deliverable | Where |
|---|---|
| Automated multi-cloud deployment system | repos `gcp-platform`, `azure-platform`, `aws-platform`, `global-platform`, `platform-deployments`, `platform-automation` |
| High availability and load balancing | Cloudflare LB in `global-platform/loadbalancer.tf`; failover wake-up in `global-platform/failover.tf` + `platform-automation/.github/workflows/failover-wake.yml` |
| Disaster recovery and data replication | `platform-deployments/pipelines/{backup,restore-drill}.Jenkinsfile`, `scripts/dr/`; [chapter 08](08-disaster-recovery-and-failover.md) |
| Monitoring and cost dashboard | `observability-platform` (dashboards + alert rules), central Grafana; [chapter 07](07-monitoring-and-cost.md) |
| Comprehensive documentation | this handbook (setup, usage, DR, troubleshooting) |

## Design decisions (and why)

| Decision | Why | Trade-off |
|---|---|---|
| **Primary + stopped standbys** instead of three always-on clouds | Standby nodes cost nothing while asleep (≈ US $212/month of node cost saved, see [cost](07-monitoring-and-cost.md)) but the configuration is complete, so waking is minutes, not a rebuild | A real outage costs a few minutes (measured RTO 3 min 42 s) instead of ~60 s |
| **Cloudflare** in front instead of one cloud's DNS | A cloud-neutral front door cannot fail together with one of the clouds | One more vendor; Load Balancing is a paid add-on |
| **One shared Atlas database** | Same data in every cloud with zero sync logic; Atlas M0 is free | The database is a shared dependency — protected by 3-cloud backups, not live replication |
| **Clusters exist only during working sessions** | Pay only for hours used; the bill for October was ≈ US $17 | Start-up takes ~25–40 minutes (parallel `up.yml`) |
| **Jenkins for CD, GitHub Actions for CI** | GitHub is good at build/test/publish per repo; Jenkins coordinates a cross-cloud rollout with gates and rollback | Jenkins lives in GKE, so a GCP outage needs the break-glass `deploy.yml` |
| **Keyless everywhere** | No secret to leak, rotate or forget | OIDC trust rules must be exact (repo id, branch) |
