# 08 — CI → CD: from a merged pull request to all three clouds

## What happens when you merge
```
merge PR on a service repo (main)
  │
  ├─ GitHub Actions (core-cloud-platform @v2)
  │    1. build + unit tests (coverage gate)
  │    2. docker build + Trivy scan (blocks fixable HIGH/CRITICAL)
  │    3. log in to AWS, Azure, GCP WITHOUT keys (OIDC)
  │    4. push the SAME image (same digest) to ECR, ACR, Artifact Registry
  │    5. "Ask Jenkins to deploy"  ──► https://jenkins.rishavmukhopadhyay.store
  │                                       lock 1: Cloudflare Access (service token)
  │                                       lock 2: Jenkins trigger token
  └─ Jenkins deploy-mern: GCP → Azure → AWS
       each cloud: Helm rolling update (--rollback-on-failure) + smoke test
       a failure rolls that cloud back and STOPS - later clouds keep the old version
```
Pull requests only do steps 1–2. Nothing is pushed or deployed from a PR.

## The pieces and where they live
| Piece | Repo / file | Purpose |
|---|---|---|
| Push identities | `aws/azure/gcp-platform/environments/prod/foundation/ci-push.tf` | Keyless "robots" that may only ADD images |
| Image workflow | `core-cloud-platform/.github/workflows/docker-build.yaml` | Build, scan, push to 3 registries (main only) |
| Deploy trigger | `core-cloud-platform/.github/workflows/trigger-deploy.yaml` | Reads 3 secrets from GCP Secret Manager, calls Jenkins |
| Language pipelines | `node-app.yaml`, `java-app.yaml`, `python-app.yaml` | Inputs `push-images`, `deploy-after-push` |
| Service opt-in | `<service>/.github/workflows/main.yaml` | `push-images: true`, `deploy-after-push: true` (not seed-db) |
| Jenkins door | `global-platform/jenkins-access.tf` | DNS (only while GKE runs), Access app + policies, service token, CI secrets |
| Jenkins side | `gcp-platform/environments/prod/addons/values/jenkins.yaml` + `charts/platform-config/templates/jenkins.yaml` | Public route, Generic Webhook Trigger, token from the vault |

## Least privilege: who may do what
| Identity | Trusted for | Allowed to |
|---|---|---|
| AWS `gha-push-<service>` (one per service) | main branch of THAT service repo | push to that service's ECR repository only |
| Azure `id-gha-push` | main branch of the 7 service repos | `AcrPush` on our registry |
| GCP `gha-push` | main branch of the 7 service repos | `artifactregistry.writer` on `mern` + read 3 CI secrets |
| Cloudflare service token | GitHub Actions | pass Cloudflare Access for `jenkins.` |
| Jenkins trigger token | GitHub Actions | start `deploy-mern` (only the 6 deployable services) |

**Identity card format.** Our repos use GitHub's *immutable* OIDC subject:
`repo:ppmcad16a-cloud-platform@308835253/<repo>@<repo id>:ref:refs/heads/main`.
The numbers are permanent ids. A repo deleted and re-created under the same name gets a new id and can NOT inherit cloud access.
Find a repo id with `gh api repos/ppmcad16a-cloud-platform/<repo> --jq .id`.

## Versioning of the shared workflows
| Tag | Meaning |
|---|---|
| `v1` | build + scan only (old callers) |
| `v2.0.0` | push to 3 registries (breaking: callers must grant `id-token: write`) |
| `v2.1.1` | + optional deploy trigger (compatible) |
| `v2` | moves forward to the newest compatible v2.x |

Release order: **merge → `git pull` → check `git log -1` → tag**. A tag points at the commit you have checked out.

## When the clusters are off
Merges still work. The image is pushed to all 3 registries, and the deploy job ends **green with a warning**:
`Jenkins is offline … deploy later with SERVICE=x IMAGE_TAG=main-abc1234`.
At the next session, run deploy-mern by hand with that tag (Build with Parameters).

## Manual operations
- **Open Jenkins:** https://jenkins.rishavmukhopadhyay.store → Cloudflare emails you a PIN → Jenkins user `admin`, password from
  `terraform output -raw jenkins_admin_password` in `gcp-platform/environments/prod/addons`.
- **Roll back:** run deploy-mern with the previous tag.
- **Deploy to one cloud only:** set `TARGETS` (e.g. `gcp`).
- **Failure demo:** `IMAGE_TAG=gcp-only` exists only in GCP. Azure fails after up to **8 minutes** (Helm `--timeout 8m`), rolls back, and AWS is skipped. Old pods keep serving the whole time.

## Proven on 2026-10-29
- 7 services built, scanned and pushed. The digest was identical in ECR, ACR and Artifact Registry.
- Offline path: green run with a "deploy later" warning.
- Online path: merge of catalog-service PR #5 → `main-c33cefa` → Jenkins started by GitHub → running in all clouds.
- Failure path: Azure `failed` → `Rollback to 2`. AWS got no new revision.

## Troubleshooting (all of these happened to us)
| Symptom | Cause | Fix |
|---|---|---|
| `Not authorized to perform sts:AssumeRoleWithWebIdentity` | trust rule expects the old name-only subject | use the immutable subject with ids |
| PR shows "no checks", `startup_failure` | caller passes an input the referenced `@v2` does not know yet | merge + tag core first, then close/reopen the PR |
| `base branch policy prohibits the merge` right after `gh pr create` | checks not registered yet | `gh pr merge --squash --delete-branch --auto` |
| Trivy fails on a new CVE | a vulnerability published after the last build | update the package (lock file / requirements / `apk upgrade`), never disable the gate |
| `awk: command not found` in Jenkins | `azure-cli` image has no awk | use `grep`/`cut`/`tr` |
| Token visible in a Jenkins log | Jenkins runs `sh` with `set -x` | `set +x` before commands that contain tokens; delete the build |
| Helm `context deadline exceeded`, then `cannot re-use a name` | slow image pull (ghcr.io on AKS) left a half-created release | delete stuck pods, `helm uninstall <name> -n <ns>`, apply again |
| `docker login` to ECR gives `400` on Windows PowerShell | PS 5.1 pipe re-encodes the token | `cmd /c "aws ecr get-login-password ... \| docker login ... --password-stdin ..."` |

## Known limitations / next steps
- Third-party images are pulled from ghcr.io / Docker Hub at deploy time. Plan: mirror them into our own registries.
- The Terraform CI logins of the platform repos still use the name-only subject (fix together with the Terraform CI workflow).
- Jenkins state is rebuilt from code each session (no disk), so build history is lost at teardown by design.
