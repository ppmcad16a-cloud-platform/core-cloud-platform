# Prerequisites: tools, accounts and logins

> **In plain words:** to build or operate the platform you need (1) the command-line tools that talk to
> each cloud, (2) accounts with permission to create things, and (3) to log in once on your computer.

## 1. Tools to install (Windows, one-time)

Open **PowerShell as Administrator** and run:

```powershell
winget install --id GitHub.cli          -e   # gh: talk to GitHub (create repos, settings)
winget install --id Hashicorp.Terraform -e   # terraform: "infrastructure as code"
winget install --id Amazon.AWSCLI       -e   # aws: talk to AWS
winget install --id Microsoft.AzureCLI  -e   # az: talk to Azure
winget install --id Google.CloudSDK     -e   # gcloud: talk to Google Cloud
winget install --id Helm.Helm           -e   # helm: install apps into Kubernetes
```

Close and reopen the terminal, then check that each one answers with a version:

```powershell
gh --version; terraform -version; aws --version; az version; gcloud --version; helm version
```

Also needed: Git, Docker Desktop (includes `kubectl`), JDK 21, Node.js 22 and Python 3.12. For the Ansible
playbooks: WSL Ubuntu 24.04 (`wsl --install -d Ubuntu-24.04`) — `platform-automation`'s `toolchain.yml` then
installs the pinned Linux versions of every CLI.

| Tool | What it is, in one line |
|---|---|
| **Terraform** | You write *what* infrastructure you want in files; it creates/updates/deletes it to match. |
| **kubectl** | Remote control for a Kubernetes cluster. |
| **Helm** | "App store / installer" for Kubernetes — installs a whole app from one package (a *chart*). |
| **aws / az / gcloud** | Each cloud's own remote control. |
| **gh** | GitHub's remote control (create repos, set branch rules, secrets). |

## 2. Tell Git who you are (one-time)

```powershell
git config --global user.name  "Your Name"
git config --global user.email "you@example.com"
```

## 3. Log in (each one opens a browser)

```powershell
gh auth login --hostname github.com --git-protocol https --web   # choose: HTTPS, "Login with a web browser"
gh auth setup-git                                                # lets plain `git push` use that login
aws configure sso    # or `aws configure` if you only have access keys (we stop using keys after bootstrap)
az login
gcloud auth login
gcloud auth application-default login   # lets Terraform use your Google login
```

Why log in as *you* only now: the very first "bootstrap" step must be done by a human admin.
That step creates **robot identities** (for GitHub Actions and Jenkins) with *only* the permissions
they need. After that, the robots do the work and nobody needs to keep admin keys lying around.

## 4. Accounts and limits to check

| Cloud | Check |
|---|---|
| AWS | Account is on the **paid plan** (credits still apply). The new "free plan" blocks EKS. |
| Azure | Note your **Subscription ID** (`az account show`). Free accounts usually allow only **4 vCPU per region** — our cluster fits in that. |
| GCP | Note your **Project ID** and that billing is linked. Trial accounts cap at **8 vCPU** total. |
| GitHub | You are **Owner** of the organisation (needed for branch protection and OIDC trust rules). |

## 5. Secrets hygiene (read once, follow forever)

- A secret (password, token, key, connection string) **never** goes into Git, chat, screenshots or tickets.
- If one leaks: **rotate it** (create a new one, delete the old one). Deleting the message is not enough.
- In this project, secrets live only in each cloud's *vault* (AWS SSM Parameter Store, Azure Key Vault,
  GCP Secret Manager) and are delivered to apps automatically.
