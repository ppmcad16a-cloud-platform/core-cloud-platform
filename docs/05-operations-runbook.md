# 05 — Operations runbook

All commands run in **Ubuntu (WSL)** inside `platform-automation`, after:

```bash
source ~/.venvs/ansible/bin/activate
cd /mnt/c/Users/dell/Documents/HeroVired/Capstone/repos/platform-automation
aws sso login --profile HeroviredProfile      # AWS SSO sessions expire - renew at the start of every session
```

Every playbook that changes Terraform goes through `tasks/tf-apply.yml`: **plan → check the kind of each change → apply exactly that plan**. Session start may only *create/update*; session end may only *delete/update*. A plan that would replace or delete something unexpectedly stops the run before anything happens.

## Quick reference

| I want to… | Command | Time |
|---|---|---|
| start a session (standbys go to sleep at the end) | `ansible-playbook playbooks/up.yml` | 25–40 min |
| start with all three clouds awake | `ansible-playbook playbooks/up.yml -e keep_awake=true` | 25–40 min |
| put the standbys to sleep | `ansible-playbook playbooks/standby.yml` (or `-e cloud=azure`) | ~5 min |
| wake one standby | `ansible-playbook playbooks/wake.yml -e cloud=aws` | 3–10 min |
| deploy without Jenkins (break-glass) | `ansible-playbook playbooks/deploy.yml -e service=catalog-service -e tag=main-ba5cc42` | ~3 min |
| prove the backups restore | `ansible-playbook playbooks/drill-restore.yml` | ~10 min |
| failover drill (GCP fails) | `ansible-playbook playbooks/drill-failover.yml` (`-e wake_via=manual` to wake by playbook) | 25–30 min |
| end the session — delete everything that costs per hour | `ansible-playbook playbooks/down.yml` | 30–40 min |

Tip: wrap long runs as `time ansible-playbook … 2>&1 | tee ~/run-$(date +%F).log` to keep a log and the duration.

## 1. Start a session

1. Make sure your public IP is in the six `terraform.tfvars` files (`admin_cidr`) — preflight checks it.
2. Type the provider secrets once (hidden, memory only):
   ```bash
   read -rsp "Cloudflare API token: " CLOUDFLARE_API_TOKEN; echo; export CLOUDFLARE_API_TOKEN
   read -rsp "Atlas public key: " MONGODB_ATLAS_PUBLIC_KEY; echo; export MONGODB_ATLAS_PUBLIC_KEY
   read -rsp "Atlas private key: " MONGODB_ATLAS_PRIVATE_KEY; echo; export MONGODB_ATLAS_PRIVATE_KEY
   ```
3. `ansible-playbook playbooks/up.yml`. Order: preflight → ACR mirrors → **GKE + AKS + EKS in parallel** → global layer → each cloud independently: add-ons → shop (6 services at once) → health check → standbys sleep.
4. Result: `https://<domain>` served by GCP; Azure stopped, AWS at 0 nodes.

The versions deployed come from `C:\Users\dell\image-tags.json`; keep it updated after each release (Jenkins records what it deployed in Slack).

## 2. End a session

```bash
ansible-playbook playbooks/down.yml
```

Order: preflight → wake sleeping standbys (so Kubernetes can remove its own load balancers and disks) → AKS + EKS: add-ons destroy → **state must be empty** → **leftover check** (no `k8s-*` load balancer, no orphaned disk) → cluster destroy → GKE the same → global layer (delete/update only).

If the leftover check stops the run, Kubernetes did not finish cleaning up. Wait 5 minutes and re-run, or remove the leftover by hand (section 7). **Never** continue with `-refresh=false`: it can wipe state while the real resources keep running.

Afterwards, prove nothing is billing:

```bash
gcloud container clusters list; gcloud compute instances list; gcloud compute forwarding-rules list
az aks list -o table
aws eks list-clusters --region ap-south-1 --profile HeroviredProfile
aws elbv2 describe-load-balancers --region ap-south-1 --profile HeroviredProfile --query "LoadBalancers[].LoadBalancerName"
aws ec2 describe-nat-gateways --region ap-south-1 --profile HeroviredProfile --filter Name=state,Values=available --query "NatGateways[].NatGatewayId"
```

Only the permanent foundation remains (registries, vaults, backup buckets, state buckets, GCP's `prod-egress` IP).

## 3. Deploy, roll back

| Situation | What to do |
|---|---|
| normal release | merge the pull request — CI pushes the image and Jenkins deploys GCP → Azure → AWS ([CI/CD](06-ci-cd.md)) |
| roll back | Jenkins → `deploy-mern` → *Build with Parameters* → the **previous** `IMAGE_TAG` |
| deploy to some clouds only | same, with `TARGETS=gcp` (comma list) |
| Jenkins is down (e.g. GCP outage) | `ansible-playbook playbooks/deploy.yml -e service=<svc> -e tag=<tag>` — deploys to every **awake and reachable** cloud with the same rolling update + rollback; `-e target=azure` for one cloud |
| an image reached only some registries | the CI run is green with a warning; re-run the workflow when the cloud is back (it fills the gap and then deploys), or deploy now with `TARGETS=` the clouds that have it |

A sleeping standby is skipped by Jenkins and caught up automatically when it wakes (`sync-shop` compares its versions with GCP's).

## 4. Standbys

| Action | Command | What happens |
|---|---|---|
| sleep | `playbooks/standby.yml` | Azure: `az aks stop`. AWS: node group min/desired 0. |
| wake | `playbooks/wake.yml -e cloud=<azure\|aws>` | start → nodes Ready → shop healthy → catch up to GCP's versions |
| automatic wake | nothing — Cloudflare notification → Worker → GitHub Actions `failover-wake.yml` | runs only when the **gcp** pool becomes unhealthy |

**Known behaviour:** AWS sleep can take up to 30 minutes because the PodDisruptionBudgets (`minAvailable: 1`) refuse to evict the last copy of each service while the whole node group is going away. The playbook reports "asleep" as soon as the *setting* is 0. Finish the drain at once:

```bash
kubectl --context arn:aws:eks:ap-south-1:722867460821:cluster/prod-eks drain -l eks.amazonaws.com/nodegroup=default \
  --ignore-daemonsets --delete-emptydir-data --disable-eviction --force --timeout=180s
```

Also wait until a woken AKS has finished starting (`az aks show … --query provisioningState` = `Succeeded`) before stopping it again — Azure refuses a stop during a start.

## 5. Drills

| Drill | Command | Pass criteria |
|---|---|---|
| restore | `playbooks/drill-restore.yml` | Jenkins `restore-drill` runs for `gcs`, `s3`, `azure`; each downloads the newest backup, verifies its SHA-256, decrypts it, restores into `mern_restore_drill` and matches document counts → `Restore drill PASSED` |
| failover | `playbooks/drill-failover.yml` | standbys asleep → GCP's gateway scaled to 0 → shop fails → standbys woken (automatically or `-e wake_via=manual`) → shop answers again → GCP repaired → standbys back to sleep → RTO ≤ 20 min |

## 6. Your public IP changed mid-session

Symptoms: `kubectl` times out (`dial tcp …:443: i/o timeout`), Atlas answers `IP_ADDRESS_NOT_ON_ACCESS_LIST`. Three places need the new IP:

1. The six `terraform.tfvars` (`admin_cidr`) — PowerShell:
   ```powershell
   $R="C:\Users\dell\Documents\HeroVired\Capstone\repos"; $old="<old-ip>/32"; $new="<new-ip>/32"
   foreach ($c in "gcp","azure","aws") { foreach ($s in "cluster","addons") {
     $f="$R\$c-platform\environments\prod\$s\terraform.tfvars"
     [IO.File]::WriteAllText($f, (Get-Content $f -Raw).Replace($old,$new)) } }
   ```
2. Atlas → Organization → Access Manager → Applications → API Keys → your key → *Edit* → API access list → *Use current IP*.
3. Re-run `up.yml` (or, before a teardown, `terraform apply` in the three `cluster` stacks — each plan must show `1 to change` on the cluster's API allow-list).

After a network change also re-test WSL downloads ([Troubleshooting](10-troubleshooting.md#wsl-downloads-time-out-after-30-s)).

## 7. Remove a leftover AWS load balancer by hand

```bash
A="--region ap-south-1 --profile HeroviredProfile"
LB=$(aws elbv2 describe-load-balancers $A --query "LoadBalancers[?starts_with(LoadBalancerName,'k8s-')].LoadBalancerArn" --output text)
TGS=$(aws elbv2 describe-target-groups $A --load-balancer-arn $LB --query "TargetGroups[].TargetGroupArn" --output text)
aws elbv2 delete-load-balancer $A --load-balancer-arn $LB && aws elbv2 wait load-balancers-deleted $A --load-balancer-arns $LB
for tg in $TGS; do aws elbv2 delete-target-group $A --target-group-arn $tg; done
# then the controller's security groups (k8s-traffic-*, k8s-envoygat-*): remove rules that reference them, then delete them
```

## 8. Rotate a secret

| Secret | How |
|---|---|
| Atlas DB passwords, JWT secret, seed admin password | `terraform apply -replace=random_password.<name>` (or `-replace` the Atlas user) in `global-platform`; External Secrets refreshes within 1 h — restart the deployments to pick it up at once |
| Slack webhook | new value in GCP Secret Manager `mern-slack-webhook-url`, then `up.yml` / `global-platform` apply copies it |
| Cloudflare / Atlas provider keys | create new ones in the provider console, revoke the old ones; they are typed in per session and never stored |
| failover GitHub token | new fine-grained PAT → `gcloud secrets versions add failover-github-token` → apply `global-platform` |
| backup age key | new key pair; keep the **old private key** until every backup encrypted with it has expired |
