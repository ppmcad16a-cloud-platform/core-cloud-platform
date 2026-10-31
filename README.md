# core-cloud-platform

The shared delivery standard of the MERN multi-cloud platform:

1. **Reusable GitHub Actions workflows** — every service repo contains only a thin caller; the real build, test, scan, image-publishing and deploy-trigger logic lives here once.
2. **The project handbook** in [`docs/`](docs/README.md) — architecture, security, setup guide, operations runbook, CI/CD, monitoring and cost, disaster recovery, the final test report and troubleshooting.

## Reusable workflows

All workflows run on `workflow_call`, use `permissions: contents: read` (plus `id-token: write` only where a cloud login is needed), pin every third-party action to a **commit SHA**, and download scanners from official releases with **checksum verification**.

| Workflow | What it does |
|---|---|
| `node-app.yaml` | `npm ci` → `npm test` (the repo's coverage gate) → build → optional image job → optional deploy trigger |
| `java-app.yaml` | one `mvn verify` (compile, tests, JaCoCo gate, package) → JAR handed to the image job → optional deploy trigger |
| `python-app.yaml` | `pip install -r requirements-dev.txt` → `pytest` → optional image job → optional deploy trigger |
| `docker-build.yaml` | Buildx build with layer cache → **Trivy** (fails on fixable HIGH/CRITICAL) → on `main`: keyless login to AWS, Azure and GCP and push the **same image** to ECR, ACR and Artifact Registry; tolerates one registry being down |
| `trigger-deploy.yaml` | reads the Cloudflare Access token and Jenkins trigger token from GCP Secret Manager and starts Jenkins `deploy-mern`; ends green with a "deploy later" warning when Jenkins is offline |
| `security-scan.yaml` | **gitleaks** over the full git history |
| `ci.yaml` | CI for this repository: `actionlint` (incl. shellcheck) on every workflow + secret scan |

### How a service uses them

```yaml
# <service>/.github/workflows/main.yaml
name: Catalog Service CI
on:
  push:
    branches: [main]
  pull_request:
permissions:
  contents: read
jobs:
  build:
    permissions:
      contents: read
      id-token: write          # the image job logs in to the clouds (main branch only)
    uses: ppmcad16a-cloud-platform/core-cloud-platform/.github/workflows/node-app.yaml@v2
    with:
      node-version: "22"
      build-docker: true
      image-name: catalog-service
      push-images: true
      deploy-after-push: true
  secret-scan:
    uses: ppmcad16a-cloud-platform/core-cloud-platform/.github/workflows/security-scan.yaml@v2
```

Pull requests only build, test and scan. Pushes to `main` additionally publish `<image>:main-<sha>` and trigger the deployment.

### Versions

Callers reference **`@v2`**, a tag that is moved forward only to tested, backward-compatible releases. A breaking change gets a new major tag.

| Tag | Content |
|---|---|
| `v1` | build, test, image build + Trivy |
| `v2.0.0` | push to ECR, ACR and Artifact Registry with keyless OIDC logins |
| `v2.1.1` | optional deploy trigger (`trigger-deploy.yaml`) |
| `v2.2.0` | registry-outage tolerance; new output `pushed-to` |

Release: merge → `git pull` → `git tag v2.x.y` → move `v2` → push both tags.

## Dependencies

| Depends on | For |
|---|---|
| [gcp-platform](https://github.com/ppmcad16a-cloud-platform/gcp-platform), [azure-platform](https://github.com/ppmcad16a-cloud-platform/azure-platform), [aws-platform](https://github.com/ppmcad16a-cloud-platform/aws-platform) | the keyless push identities and registries (`environments/prod/foundation/ci-push.tf`) trusted for each service repo's `main` branch |
| [global-platform](https://github.com/ppmcad16a-cloud-platform/global-platform) | the CI secrets in GCP Secret Manager and the Cloudflare Access application in front of Jenkins |
| Jenkins in GKE ([gcp-platform](https://github.com/ppmcad16a-cloud-platform/gcp-platform) add-ons) | receives the deploy trigger |

| Used by | |
|---|---|
| [api-gateway-service](https://github.com/ppmcad16a-cloud-platform/api-gateway-service), [identity-service](https://github.com/ppmcad16a-cloud-platform/identity-service), [catalog-service](https://github.com/ppmcad16a-cloud-platform/catalog-service), [commerce-service](https://github.com/ppmcad16a-cloud-platform/commerce-service), [notification-service](https://github.com/ppmcad16a-cloud-platform/notification-service), [client-service](https://github.com/ppmcad16a-cloud-platform/client-service), [seed-db-service](https://github.com/ppmcad16a-cloud-platform/seed-db-service) | every CI run |

## Documentation

Start with the [handbook](docs/README.md). The [setup guide](docs/04-setup-guide.md) explains how to run the shop locally or rebuild the whole platform in your own accounts; background guides are in [`docs/guides/`](docs/guides/).

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).
