# 10 — Troubleshooting

Every entry below happened to us. Format: **symptom → cause → fix**.

## Sessions (`up.yml` / `down.yml`) and WSL

### WSL downloads time out after 30 s
**Symptom:** `terraform plan/destroy` of an add-ons stack fails with `Error locating chart … release-assets.githubusercontent.com … context deadline exceeded (Client.Timeout exceeded while awaiting headers)`; only some runs fail.
**Cause:** one of GitHub's four download addresses (`185.199.109.133`) was unreachable from the home ISP. Each download picks an address at random; Helm's client waits 30 s on the dead one. Windows browsers hide the problem by retrying the next address quickly.
**Diagnose:**
```bash
for ip in 185.199.108.133 185.199.109.133 185.199.110.133 185.199.111.133; do
  curl -s -o /dev/null -w "$ip %{time_connect}s rc=%{exitcode}\n" --max-time 8 \
    --resolve release-assets.githubusercontent.com:443:$ip https://release-assets.githubusercontent.com/; done
```
**Fix:** pin a working address for the session: `echo "185.199.108.133 release-assets.githubusercontent.com objects.githubusercontent.com raw.githubusercontent.com" | sudo tee -a /etc/hosts` (WSL rewrites `/etc/hosts` at restart). *Lesson:* we first suspected WSL's DNS relay — measuring connect time per address found the real cause.

### AWS steps fail with `refresh cached SSO token failed … InvalidRequestException`
**Cause:** the AWS SSO session expired. `aws sts get-caller-identity` can still pass on cached short-term keys while Terraform cannot renew them, so a failed host drops out of the play and the global layer fails later.
**Fix:** `aws sso login --profile HeroviredProfile` at the start of every session, then re-run the playbook.

### Your IP changed mid-session — `kubectl` `dial tcp …:443: i/o timeout`, Atlas `IP_ADDRESS_NOT_ON_ACCESS_LIST`
**Cause:** home/hotspot IPs change; the cluster APIs and the Atlas API key only accept the old one.
**Fix:** [Runbook §6](05-operations-runbook.md#6-your-public-ip-changed-mid-session) — update the six `terraform.tfvars`, the Atlas API key access list, then re-apply the cluster stacks.

### `down.yml` stops: "Something the add-ons created is still there (k8s-envoygat-…)"
**Cause:** the AWS Load Balancer Controller was removed before it deleted Envoy's NLB. The safety stop keeps the cluster so nothing is orphaned.
**Fix:** wait 5 min and re-run, or delete the NLB, its target group and the two `k8s-*` security groups by hand ([Runbook §7](05-operations-runbook.md#7-remove-a-leftover-aws-load-balancer-by-hand)).

### AWS standby "asleep" but EC2 instances keep running for up to 30 min
**Cause:** scaling the node group to 0 drains the nodes; PodDisruptionBudgets (`minAvailable: 1`) refuse to evict the last copy while every node is going away; the termination hook times out after 30 min.
**Fix:** `kubectl drain -l eks.amazonaws.com/nodegroup=default --disable-eviction --force --ignore-daemonsets --delete-emptydir-data`.

### `az aks stop` → `OperationNotAllowed … in-progress start managed cluster operation`
**Cause:** a standby woken by the failover drill was still starting when the drill put it back to sleep.
**Fix:** wait for `provisioningState = Succeeded`, then stop.

### Never run `terraform destroy -refresh=false` on add-ons
**What happened:** with a stale GKE token the Helm provider skipped the uninstalls but the state was emptied — releases kept running with no state.
**Fix:** restored the previous state version from the versioned bucket (`gcloud storage cp "<obj>#<generation>" <obj>`) and destroyed normally. State buckets are versioned exactly for this.

### Global teardown blocked by an "update" of `cloudflare_notification_policy_webhooks.slack`
**Cause:** the provider plans an update on every run (read-only fields come back empty).
**Fix:** the teardown guard allows `[delete, update]` for the global stack — never create or replace.

### Ansible `ANSIBLE_CONFIG` ignored / play names with inventory variables fail / strict booleans
**Cause:** an old export in `~/.bashrc`; Ansible 2.21 does not template play names with host variables and requires real booleans in `when`/`until`.
**Fix:** `sed -i '/ANSIBLE_CONFIG/d' ~/.bashrc`; plain-text play names; write conditions like `x is defined and x is not none`.

## CI/CD

| Symptom | Cause | Fix |
|---|---|---|
| `Not authorized to perform sts:AssumeRoleWithWebIdentity` | trust expects GitHub's immutable subject (org id + repo id) | use `repo:<org>@<org-id>/<repo>@<repo-id>:ref:refs/heads/main` |
| PR shows "no checks" / `startup_failure` | caller passes an input the referenced `@v2` does not know yet | merge + tag `core-cloud-platform` first, then re-run |
| `base branch policy prohibits the merge` right after `gh pr create` | checks not registered yet | `gh pr merge --squash --delete-branch --auto` |
| Trivy fails on a new CVE | vulnerability published after the last build | update the package; never disable the gate |
| `docker login` to ECR gives 400 in Windows PowerShell 5.1 | PS 5.1 pipe re-encodes the token | `cmd /c "aws ecr get-login-password … \| docker login … --password-stdin …"` |
| Jenkins `awk: command not found` | `azure-cli` image has no awk | use `grep`/`cut`/`tr` |
| a token visible in a Jenkins log | `sh` runs with `set -x` | `set +x` before commands that use tokens; delete the build |
| Helm `context deadline exceeded`, then `cannot re-use a name` on AKS | images from ghcr.io pulled very slowly in Azure | mirror the images into ACR (`up.yml` does it) |
| failover Worker gets 403 from GitHub | fine-grained PAT waiting for org-owner approval | approve it in the org settings |

## Data and security

| Symptom | Cause | Fix |
|---|---|---|
| Atlas IP access list contains `0.0.0.0/0` | added by hand in early local development | removed on 31 Oct 2026; only Terraform-managed cluster IPs should remain |
| backup upload to Azure: `403` on HEAD | rclone reads before writing; the identity is write-only | upload with two REST `Put Blob` calls (`upload-azure.sh`) |
| backup to GCS: bucket check refused | rclone tries to create/check the bucket | `RCLONE_CONFIG_GCS_NO_CHECK_BUCKET=true` |
| orphaned AWS EBS volumes after teardown | the EBS CSI policy was detached before the volumes were deleted | `depends_on` on the policy attachment + leftover check in `down.yml` |
| `terraform init -upgrade` pulled a brand-new provider | upgrade ignores the lock file | always plain `terraform init`; adopt versions ≥ 14 days old |
