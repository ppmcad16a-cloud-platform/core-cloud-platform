# Contributing

These rules apply to every repository of the organisation.

## Changes reach `main` only through pull requests

```bash
git switch main && git pull
git switch -c <type>/<short-description>      # e.g. fix/catalog-pagination
# edit, test locally
git commit -m "<type>: <what and why>"         # types: feat, fix, docs, chore, refactor, test
git push -u origin <branch>
gh pr create --fill
gh pr merge --squash --delete-branch --auto    # merges when the required checks are green
```

`main` is protected (admins included): no direct pushes, no force-pushes, required checks must pass.

## Required checks

| Repository type | Checks |
|---|---|
| service repos | build + tests with the coverage gate, Docker build + Trivy, gitleaks |
| core-cloud-platform | actionlint + shellcheck on every workflow, gitleaks |

## Rules

* **No secrets in git** — not in code, `terraform.tfvars`, examples or comments. Secrets live in the cloud vaults; provider tokens are typed in per session.
* Pin third-party GitHub Actions to a full commit SHA and adopt new tool, provider or chart versions only once they are at least 14 days old.
* Terraform: plain `terraform init` (never `-upgrade`), commit `.terraform.lock.hcl`, run `terraform fmt` and `terraform validate` before opening the PR, and read the plan before applying.
* Shared workflows: a backward-compatible change moves the `v2` tag after it is tested; a breaking change needs a new major tag.
* Keep the README of the repository you change accurate: purpose, usage, dependencies.
