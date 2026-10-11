# 01 — Repositories and shared CI pipelines

> **In plain words:** a microservice company keeps **one Git repository per service** so that a
> change to the catalog only rebuilds and redeploys the catalog. To stop every team writing its
> own (slightly different) pipeline, the pipeline is written **once** in a shared repository and
> every service just *calls* it.

## 1. How the organisation is laid out

| Kind | Repositories | What lives there |
|---|---|---|
| Services | `api-gateway-service`, `identity-service`, `catalog-service`, `commerce-service`, `notification-service`, `client-service`, `seed-db-service` | App code, tests, Dockerfile, a tiny `.github/workflows/main.yaml` |
| Shared pipelines + docs | `core-cloud-platform` | Reusable GitHub Actions workflows, these guides |
| Infrastructure | `terraform-modules`, `aws-platform`, `azure-platform`, `gcp-platform`, `global-platform` | Terraform code (later phases) |
| Runtime / ops | `platform-deployments`, `observability-platform`, `security-platform` | Helm charts, Jenkins, monitoring, policies (later phases) |

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
jobs.build.uses: .../node-app.yaml@v1                 steps: checkout → npm ci → npm test → build
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

## 4. Versioning the shared workflows (`@v1`)

Services call `...@v1`, not `...@main`.

- Changes to `core-cloud-platform` go through a pull request; CI lints every workflow (actionlint).
- After merge and a test, the **`v1` tag is moved** to the new commit:
  ```bash
  git tag -f v1 && git push -f origin v1
  ```
- A change that would break callers gets a **new** tag `v2`; services move when ready.

## 5. Redo it yourself — checklist

1. Create the shared repo, put workflows in `.github/workflows/` with `on: workflow_call`.
2. Pin every `uses:` to a SHA: `gh api repos/<owner>/<action>/commits/<tag> --jq .sha`.
3. Give every workflow `permissions: contents: read` (add more only where needed).
4. Lint locally: download `actionlint` from its GitHub release, run `actionlint` in the repo.
5. Tag a release: `git tag v1 && git push origin v1`.
6. In each service repo add a thin caller (`uses: <org>/<repo>/.github/workflows/<file>@v1`).
7. Open a pull request — the PR run is your test. Merge when green.

## 6. Lessons learned during this phase

- **Always list an organisation's repos while logged in** (`gh repo list <org>`). Logged-out
  requests hide private repos — we briefly created duplicates of repos that already existed.
- GitHub refuses a push that would overwrite someone else's history ("non-fast-forward"). That is
  a safety feature: investigate, don't force-push.
