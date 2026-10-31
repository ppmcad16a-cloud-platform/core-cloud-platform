# Repositories and shared CI pipelines

> **In plain words:** a microservice company keeps **one Git repository per service** so that a
> change to the catalog only rebuilds and redeploys the catalog. To stop every team writing its
> own (slightly different) pipeline, the pipeline is written **once** in a shared repository and
> every service just *calls* it.

## 1. How the organisation is laid out

| Kind | Repositories | What lives there |
|---|---|---|
| Services | `api-gateway-service`, `identity-service`, `catalog-service`, `commerce-service`, `notification-service`, `client-service`, `seed-db-service` | App code, tests, Dockerfile, a tiny `.github/workflows/main.yaml` |
| Shared pipelines + docs | `core-cloud-platform` | Reusable GitHub Actions workflows, the project handbook |
| Infrastructure | `gcp-platform`, `azure-platform`, `aws-platform`, `global-platform` | Terraform for each cloud and for the cross-cloud layer (Cloudflare, Atlas, secrets) |
| Runtime / operations | `platform-deployments`, `observability-platform`, `platform-automation` | Helm chart + Jenkins pipelines, alert rules + dashboards, Ansible playbooks |
| Organisation | `.github` | Organisation profile page |

All repositories are **public**. On a free GitHub organisation that unlocks branch protection,
deployment approval gates and unlimited Actions minutes. That is safe **only because secrets never
live in Git** — they live in each cloud's vault.

## 2. What a "reusable workflow" is

A normal workflow runs on events (`push`, `pull_request`). A *reusable* workflow runs on
`workflow_call`, which means "another workflow may call me like a function, passing inputs".

```
catalog-service/.github/workflows/main.yaml          core-cloud-platform/.github/workflows/node-app.yaml
--------------------------------------------         -------------------------------------------------
on: push / pull_request                    ───────►   on: workflow_call (inputs: node-version, ...)
jobs.build.uses: .../node-app.yaml@v2                 steps: checkout → npm ci → npm test → build
  with: node-version: "22"                            optional: docker-build.yaml (build + Trivy)
```

## 3. Security practices used (and why)

| Practice | Why |
|---|---|
| **Pin actions to a commit SHA** (`actions/checkout@3d3c42e…  # v7.0.1`) | A tag like `v4` can be moved to different code by whoever controls that repo. A SHA can never change. In March 2026 attackers moved tags of a popular scanner action — SHA-pinned users were safe. |
| **Wait ~2 weeks before adopting a new release** | Malicious releases are usually discovered within days. |
| **Download scanners from official releases and verify the checksum** | Same idea for tools: if the file was tampered with, `sha256sum -c` fails and the job stops. |
| **`permissions: contents: read`** | The job's GitHub token can only read code — even a compromised step cannot push code or change settings. |
| **`persist-credentials: false`** on checkout | The token is not left lying in `.git/config` for later steps. |
| **Trivy gate** | Build fails if the image has HIGH/CRITICAL vulnerabilities **that already have a fix**. |
| **gitleaks over the whole history** | A secret deleted in a later commit is still readable in the old commit. |
| **`.gitignore` blocks `.env`, `*.pem`, `*.key`** | Prevents committing local secrets by accident. |

## 4. Versioning the shared workflows (`@v2`)

Services call `...@v2`, not `...@main`.

- Changes to `core-cloud-platform` go through a pull request; CI lints every workflow (actionlint).
- After merge, the release gets its own tag (`v2.2.0`) and the **`v2` tag is moved** to it:
  ```bash
  git tag v2.2.0 && git push origin v2.2.0
  git tag -f v2 && git push -f origin v2
  ```
- A change that would break callers gets a **new** major tag (`v3`); services move when ready.
- History: `v1` build + scan · `v2.0.0` push to 3 registries · `v2.1.1` deploy trigger · `v2.2.0` registry-outage tolerance.

## 5. Redo it yourself — checklist

1. Create the shared repo, put workflows in `.github/workflows/` with `on: workflow_call`.
2. Pin every `uses:` to a SHA: `gh api repos/<owner>/<action>/commits/<tag> --jq .sha`.
3. Give every workflow `permissions: contents: read` (add more only where needed).
4. Lint locally: download `actionlint` from its GitHub release, run `actionlint` in the repo.
5. Tag a release: `git tag v2 && git push origin v2`.
6. In each service repo add a thin caller (`uses: <org>/<repo>/.github/workflows/<file>@v2`).
7. Open a pull request — the PR run is your test. Merge when green.

## 6. Lessons learned

- **Always list an organisation's repos while logged in** (`gh repo list <org>`). Logged-out
  requests hide private repos, which makes it easy to create duplicates by mistake.
- GitHub refuses a push that would overwrite someone else's history ("non-fast-forward"). That is
  a safety feature: investigate, don't force-push.

## 7. Branch protection (how changes reach `main`)

Every repository protects its `main` branch, **admins included**:

- No direct pushes: every change goes through a **pull request**.
- Service repos and `core-cloud-platform` can only merge when the pipeline checks are green
  (`build / build`, `secret-scan / Secret scan (gitleaks)`, and for images the Docker scan).
- No force-push, no deleting `main`.

Making a change:

```bash
git checkout -b my-change          # work on a branch
git push -u origin my-change
gh pr create                       # open the pull request, wait for green checks
gh pr merge --squash --delete-branch
```

Why: `main` is what gets deployed. Branch protection guarantees that nothing reaches it without
having passed tests, coverage, the vulnerability scan and the secret scan.