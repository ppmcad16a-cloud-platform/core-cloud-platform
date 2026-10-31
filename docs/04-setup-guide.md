# 04 — Setup guide: run it yourself

There are two ways to run this project:

| Option | What you get | Needs | Time | Cost |
|---|---|---|---|---|
| **A. Local** | the whole shop (6 services + MongoDB) on your laptop | Docker Desktop | ~30 min | free |
| **B. Full platform** | the multi-cloud platform in **your own** AWS, Azure and GCP accounts | 3 cloud accounts, a domain on Cloudflare, Atlas, Slack, a GitHub org | 1 day the first time, then ~30 min per session | ≈ US $1 per working hour while all clusters run |

Commands are marked **PowerShell** (Windows) or **Ubuntu** (WSL). Never mix them: a prompt starting with `PS C:\` is PowerShell, a prompt like `user@host:/mnt/c/...$` is Ubuntu.

---

## Option A — the shop on your laptop

**Prerequisites:** Git, Docker Desktop (4 GB RAM free), JDK 21 (only to build the Java service).

```powershell
# PowerShell — clone the services and the deployment repo side by side
mkdir mern; cd mern
foreach ($r in "api-gateway-service","identity-service","catalog-service","commerce-service","notification-service","client-service","seed-db-service","platform-deployments") {
  git clone https://github.com/ppmcad16a-cloud-platform/$r.git
}

cd platform-deployments\local
copy .env.example .env
# edit .env: JWT_SECRET (at least 32 random characters), SEED_ADMIN_EMAIL, SEED_ADMIN_PASSWORD
python -c "import secrets; print(secrets.token_urlsafe(48))"     # a good JWT_SECRET

cd ..\..\identity-service; .\mvnw.cmd -B package -DskipTests; cd ..\platform-deployments\local
docker compose up --build -d                    # first build takes several minutes
docker compose ps                               # wait until every service is "healthy"
docker compose --profile seed run --rm seed     # once: admin user + 100 sample products
```

Open **http://localhost:8080**. Check the backend with `curl http://localhost:3000/api/health` — every service should say `"up"`. Stop with `docker compose down` (add `-v` to delete the local database). Details: `platform-deployments/local/README.md`.

---

## Option B — the full multi-cloud platform in your accounts

### B1. Accounts and services you need

| Item | What exactly | Notes |
|---|---|---|
| **AWS** | an account on the paid plan (credits are fine), an SSO/admin user | the AWS free plan blocks EKS |
| **Azure** | a subscription with ≥ 4 vCPU quota in your region | free trials allow 4 vCPU per region — the design fits that |
| **GCP** | a project with billing linked | trial accounts cap at 8 vCPU |
| **Domain on Cloudflare** | a zone you own + the **Load Balancing** add-on | needed for the global front door and failover |
| **Cloudflare API token** | zone DNS/SSL/Load Balancing edit + account Access, Workers, Notifications edit | typed in at session start, never stored |
| **MongoDB Atlas** | an M0 (free) cluster, an **organisation API key** with *Project Owner* on that project, your IP on the key's access list | typed in at session start |
| **Slack** | a channel + an *incoming webhook* URL | stored once in GCP Secret Manager |
| **GitHub** | an organisation where you are Owner | branch protection + OIDC trusts are per repo |

### B2. Your copy of the repositories

Create your organisation, then copy every repository into it (history included):

```powershell
# PowerShell — needs the GitHub CLI (gh auth login first)
$org = "<your-org>"
$repos = "core-cloud-platform","api-gateway-service","identity-service","catalog-service","commerce-service","notification-service","client-service","seed-db-service","gcp-platform","azure-platform","aws-platform","global-platform","platform-deployments","observability-platform","platform-automation"
foreach ($r in $repos) {
  git clone --mirror https://github.com/ppmcad16a-cloud-platform/$r.git "$r.git"
  gh repo create "$org/$r" --public
  git -C "$r.git" push --mirror "https://github.com/$org/$r.git"
  git clone "https://github.com/$org/$r.git"
}
```

Then replace the project-specific values. Find every place with one search in the folder that holds all repos:

```powershell
Get-ChildItem -Recurse -File -Exclude *.lock.hcl | Select-String -List -Pattern "722867460821|9c0602c2|0fd463ca|project-899ced78|692591977615|rishavmukhopadhyay|ppmcad16a|0botwx|011ab039|9b10c019|6a00a473|fnonccf|308835253|rishav1994sonai" | Select-Object Path -Unique
```

| Value in this project | Meaning | Replace with |
|---|---|---|
| `722867460821`, `ap-south-1` | AWS account id, region | yours (`aws sts get-caller-identity`) |
| `9c0602c2-…`, `0fd463ca-…`, `centralindia` | Azure subscription id, tenant id, region | yours (`az account show`) |
| `project-899ced78-a554-40c2-96c`, `692591977615`, `asia-south1` | GCP project id, project **number**, region | yours (`gcloud projects describe <id>`) |
| `ppmcad16a-cloud-platform`, `308835253` | GitHub org name and org **id** | yours (`gh api orgs/<org> --jq .id`) |
| repo ids in `*/environments/prod/foundation/ci-push.tf` and `failover-identity.tf` | GitHub numeric repo ids used in OIDC trust rules | `gh api repos/<org>/<repo> --jq .id` for each repo |
| `rishavmukhopadhyay.store` | public domain | your Cloudflare zone |
| `011ab039…`, `9b10c019…` | Cloudflare zone id, account id | Cloudflare dashboard → zone overview |
| `6a00a473…`, `capstoneproject.fnonccf.mongodb.net`, `CapstoneProject` | Atlas project id, SRV host, cluster name | Atlas → Project settings / Connect |
| `0botwx`, `o398md` | random suffixes of Azure names | generated by Terraform — update the references after your foundation apply |
| `rishav1994sonai@gmail.com` | budget alert e-mail | yours |
| `AGE_RECIPIENT` in `platform-deployments/pipelines/backup.Jenkinsfile` | backup **public** key | yours (step B6) |

### B3. Workstation (Windows 11 + WSL Ubuntu 24.04)

```powershell
# PowerShell (admin), once
wsl --install -d Ubuntu-24.04
winget install Git.Git GitHub.cli Docker.DockerDesktop
```

```bash
# Ubuntu — Ansible in its own Python environment, then every pinned tool with one playbook
sudo apt-get update && sudo apt-get install -y python3-venv wslu
python3 -m venv ~/.venvs/ansible && source ~/.venvs/ansible/bin/activate
pip install "ansible-core==2.21.5" "kubernetes==37.0.1" "python-debian==1.1.1"
cd /mnt/c/<path-to-your-repos>/platform-automation
ansible-galaxy collection install -r requirements.yml
ansible-playbook playbooks/toolchain.yml -K      # terraform, kubectl, helm, kubelogin, aws, az, gcloud (checksum-verified)
```

Edit `platform-automation/inventory/group_vars/all.yml`: `repos_root` (your repos folder as seen from WSL), `image_tags_file`, the account ids, `domain`, `acr_name`.

> **Network tip:** if Helm chart downloads time out after exactly 30 s, one of GitHub's download addresses may be unreachable from your ISP — test and pin a working one as described in [Troubleshooting](10-troubleshooting.md#wsl-downloads-time-out-after-30-s).

### B4. Log in (Ubuntu)

```bash
gcloud auth login && gcloud auth application-default login
BROWSER=wslview az login --tenant <tenant-id> && az account set --subscription <subscription-id>
aws configure sso --profile HeroviredProfile          # first time; later: aws sso login --profile HeroviredProfile
```

### B5. Bootstrap each cloud (once)

The bootstrap stack creates the bucket that will hold its own state, so it is applied with local state first and then moved:

```bash
# Ubuntu — repeat for gcp-platform, azure-platform, aws-platform
cd <repos>/gcp-platform/bootstrap
mv backend.tf backend.tf.off          # 1. no remote state yet
terraform init && terraform apply     #    creates state bucket, GitHub OIDC trust, plan/apply identities, budget
mv backend.tf.off backend.tf          # 2. now move the state into the new bucket
terraform init -migrate-state         #    answer "yes"
```

### B6. One-time secrets (stored by hand, never in git)

```bash
# Ubuntu
# 1. Slack webhook -> GCP Secret Manager (global-platform copies it to the other two vaults)
printf '%s' 'https://hooks.slack.com/services/…' | gcloud secrets create mern-slack-webhook-url --data-file=-

# 2. Backup encryption key pair. The PRIVATE key goes only into the vault (and your password manager);
#    the PUBLIC key (age1…) goes into AGE_RECIPIENT in backup.Jenkinsfile.
sudo apt-get install -y age
age-keygen -o /tmp/age.key && grep 'public key' /tmp/age.key
gcloud secrets create mern-backup-age-key --data-file=/tmp/age.key && shred -u /tmp/age.key

# 3. Failover token: a fine-grained GitHub token with Actions: read & write on platform-automation only
read -rsp "PAT: " T; printf '%s' "$T" | gcloud secrets create failover-github-token --data-file=-; unset T
```

### B7. Permanent foundations

```bash
# Ubuntu — repeat for gcp-platform, azure-platform, aws-platform
cd <repos>/gcp-platform/environments/prod/foundation
terraform init && terraform plan        # read it
terraform apply                         # registry, vault, backup bucket, CI/DR/failover identities (GCP: also the egress IP)
```

Create the git-ignored inputs:

```bash
MYIP=$(curl -s https://api.ipify.org)/32
for c in gcp azure aws; do for s in cluster addons; do
  echo "admin_cidr = \"$MYIP\"" > <repos>/$c-platform/environments/prod/$s/terraform.tfvars
done; done
cat > <repos>/global-platform/terraform.tfvars <<EOF
admin_emails      = ["you@example.com"]
workers_subdomain = "<your-workers-subdomain>"
EOF
```

### B8. First images

One green run on `main` per service fills all three registries. The copy in B2 already started a run per repo, but it failed at the cloud login because the OIDC trusts did not exist yet — re-run it now:

```powershell
# PowerShell — re-run the latest main-branch run of every service repo
foreach ($r in "api-gateway-service","identity-service","catalog-service","commerce-service","notification-service","client-service","seed-db-service") {
  $id = gh run list -R "<your-org>/$r" --branch main -L 1 --json databaseId -q ".[0].databaseId"
  gh run rerun $id -R "<your-org>/$r"
}
```

Also turn on branch protection for `main` in every repo (pull request + required checks), as described in [Repositories & shared CI](guides/repositories-and-ci.md).

Write the tags they produced (`main-<sha>`, visible in each run's summary) into the file named by `image_tags_file`:

```json
{ "identity-service": "main-xxxxxxx", "catalog-service": "main-xxxxxxx", "commerce-service": "main-xxxxxxx",
  "notification-service": "main-xxxxxxx", "api-gateway-service": "main-xxxxxxx", "client-service": "main-xxxxxxx",
  "seed-db-service": "main-xxxxxxx" }
```

### B9. First session

```bash
# Ubuntu, in platform-automation — type the two provider secrets once (hidden, memory only)
read -rsp "Cloudflare API token: " CLOUDFLARE_API_TOKEN; echo; export CLOUDFLARE_API_TOKEN
read -rsp "Atlas public key: " MONGODB_ATLAS_PUBLIC_KEY; echo; export MONGODB_ATLAS_PUBLIC_KEY
read -rsp "Atlas private key: " MONGODB_ATLAS_PRIVATE_KEY; echo; export MONGODB_ATLAS_PRIVATE_KEY

time ansible-playbook playbooks/up.yml -e keep_awake=true       # 25-40 min

# once: admin user + sample catalogue (the database is shared, so run it in one cloud only)
TAG=main-xxxxxxx     # the seed-db-service tag from your image-tags file
IMG=asia-south1-docker.pkg.dev/<gcp-project>/mern/seed-db-service:$TAG
sed "s#IMAGE_PLACEHOLDER#$IMG#" ../platform-deployments/jobs/seed-db.yaml | kubectl --context <gke-context> apply -f -
kubectl --context <gke-context> -n mern wait --for=condition=complete job/seed-db --timeout=5m
```

Open `https://<your-domain>`, `https://jenkins.<your-domain>` and `https://grafana.<your-domain>` (Cloudflare Access e-mails you a PIN; passwords: `terraform output -raw jenkins_admin_password` / `grafana_admin_password` in `gcp-platform/environments/prod/addons`).

From now on every merged pull request deploys itself ([CI/CD](06-ci-cd.md)). At the end of a session run `ansible-playbook playbooks/down.yml` ([Runbook](05-operations-runbook.md)).

### B10. Checklist — is everything right?

| Check | Expected |
|---|---|
| `for c in gcp azure aws; do curl -s https://$c.<domain>/api/health; done` | `"status":"ok"` three times |
| Cloudflare → Load Balancing | 3 pools healthy (or gcp healthy + standbys critical after `standby.yml`) |
| Grafana → *MERN shop – multi-cloud overview* | one row per cloud, 6 services with ready pods |
| Jenkins | jobs `deploy-mern`, `backup-mongo`, `restore-drill` |
| `ansible-playbook playbooks/drill-restore.yml` | `Restore drill PASSED` |
