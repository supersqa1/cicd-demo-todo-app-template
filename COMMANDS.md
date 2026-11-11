# CI/CD tutorial — command reference

Use this as a lookup while following the video. We ask AI for commands on camera; this page keeps them in one place. The template includes the app and tests; we create the pipeline during the tutorial.

Replace `YOUR_OWNER`, `YOUR_REPO`, and `SERVER_IP` with your values. Run local commands from your repository folder unless stated otherwise. Windows examples use **Command Prompt (CMD)**.

## Contents

- [1. Clone and run locally](#1-clone-and-run-locally)
- [2. Log into the server and clone](#2-log-into-the-server-and-clone)
- [3. Install Docker on Ubuntu](#3-install-docker-on-ubuntu)
- [4. Run the app on the server](#4-run-the-app-on-the-server)
- [5. Run tests locally](#5-run-tests-locally)
- [6. Create and install the SSH key](#6-create-and-install-the-ssh-key)
- [7. Configure GitHub environments and images](#7-configure-github-environments-and-images)
- [8. Commit changes and inspect deployment](#8-commit-changes-and-inspect-deployment)

## 1. Clone and run locally

**GitHub:** Use this template → Create a new repository. Clone your new repository, not the template itself.

**Your computer — Mac or Windows CMD** (Docker Desktop must be running):

```sh
git clone https://github.com/YOUR_OWNER/YOUR_REPO.git
cd YOUR_REPO
docker compose up --build -d
docker compose ps
```

Open `http://localhost:5151/`. Create a task, then open `tasks/` in VS Code to see its text file. The bind mount preserves these files when containers are replaced.

```sh
docker compose logs -f
docker compose down
```

Ctrl+C exits log following without stopping the app. `down` stops the app and keeps `tasks/`.

Docker builds for its host by default: Apple Silicon → ARM64; x64 server → AMD64. Optional explicit builds:

```sh
docker buildx build --platform linux/arm64 -t cicd-demo-todo-app:arm64 --load .
docker buildx build --platform linux/amd64 -t cicd-demo-todo-app:amd64 --load .
```

These create images only; they do not replace the running Compose app. Cross-platform builds need a compatible builder/emulation, as supplied by Docker Desktop.

## 2. Log into the server and clone

**Your computer — Mac or Windows CMD:**

```sh
ssh root@SERVER_IP
```

Verify the first connection's host fingerprint using the provider console. Enter the server password; typing is invisible. This disposable demo uses root; it is not a server-hardening walkthrough.

**Inside the Ubuntu server terminal:**

```bash
whoami
cat /etc/os-release
git --version
```

If Git is missing: `apt-get update && apt-get install -y git`.

```bash
mkdir -p ~/projects
cd ~/projects
git clone https://github.com/YOUR_OWNER/YOUR_REPO.git
cd YOUR_REPO
```

A public repo needs no GitHub credentials for this HTTPS clone.

## 3. Install Docker on Ubuntu

**On a fresh Ubuntu server, logged in as root.** Check the installed OS above; use a currently supported Ubuntu release. Ask AI for instructions for that version and compare with [Docker's official Ubuntu guide](https://docs.docker.com/engine/install/ubuntu/). Existing Docker installations may need different upgrade/conflict steps.

```bash
apt-get update
apt-get install -y ca-certificates curl
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc
```

```bash
cat > /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF
apt-get update
apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
systemctl enable --now docker
docker --version
docker compose version
docker run --rm hello-world
```

Compose is included by `docker-compose-plugin`; use `docker compose` with a space.

## 4. Run the app on the server

**Server — from `~/projects/YOUR_REPO`:** Keep the app container's default non-root UID 10001 and give it write access to the task folder.

```bash
mkdir -p tasks
chown -R 10001:10001 tasks
unset LOCAL_UID LOCAL_GID
docker compose up --build -d
docker ps
docker images
curl -I http://127.0.0.1:5151/
```

Open **`http://SERVER_IP:5151/`** on your computer. Include `http` and port `5151`. If local curl works but the browser cannot connect, check the server/cloud firewall permits inbound TCP 5151 and Compose publishes `5151:5151`.

```bash
docker compose logs --tail 50
```

## 5. Run tests locally

**Mac Terminal:**

```bash
python3 -m venv .venv
source .venv/bin/activate
```

**Windows CMD** (Python 3.12 installed):

```cmd
py -3.12 -m venv .venv
.venv\Scripts\activate.bat
```

**Either platform, after activation:**

```sh
python -m pip install -r requirements-test.txt
python -m pytest tests/unit tests/integration
```

Backend tests use temporary files; no running server is needed. For browser regression, start the app first:

```sh
python -m playwright install chromium
python -m pytest tests/regression --base-url http://localhost:5151
```

Replace the base URL with `http://SERVER_IP:5151` to test staging. Chromium runs headlessly. Open `reports/report.html`; each invocation overwrites the report. Failure screenshots/traces are in `reports/playwright/`.

## 6. Create and install the SSH key

**Mac Terminal — check, then generate:**

```bash
command -v ssh-keygen
mkdir -p ~/.ssh
ssh-keygen -t ed25519 -f "$HOME/.ssh/cicd-demo-dev-deploy" -C "cicd-demo-dev-deploy"
```

**Windows CMD — check, then generate:**

```cmd
where ssh-keygen
if not exist "%USERPROFILE%\.ssh" mkdir "%USERPROFILE%\.ssh"
ssh-keygen -t ed25519 -f "%USERPROFILE%\.ssh\cicd-demo-dev-deploy" -C "cicd-demo-dev-deploy"
```

`ssh-keygen` comes with OpenSSH (built into macOS; normally available on modern Windows). If missing on Windows, run this in **administrator CMD**, reopen CMD, and check again:

```cmd
DISM /Online /Add-Capability /CapabilityName:OpenSSH.Client~~~~0.0.1.0
```

Ed25519 is a modern SSH key algorithm. Press Enter twice for an empty passphrase on this dedicated unattended deployment key. Do not overwrite an existing key. The `.pub` file goes on the server; the private file goes in an Actions secret, never Git.

**Install the public key, then test — Mac:**

```bash
cat ~/.ssh/cicd-demo-dev-deploy.pub | ssh root@SERVER_IP 'mkdir -p ~/.ssh; chmod 700 ~/.ssh; cat >> ~/.ssh/authorized_keys; chmod 600 ~/.ssh/authorized_keys'
ssh -i ~/.ssh/cicd-demo-dev-deploy -o IdentitiesOnly=yes root@SERVER_IP
```

**Windows CMD:**

```cmd
type "%USERPROFILE%\.ssh\cicd-demo-dev-deploy.pub" | ssh root@SERVER_IP "mkdir -p ~/.ssh; chmod 700 ~/.ssh; cat >> ~/.ssh/authorized_keys; chmod 600 ~/.ssh/authorized_keys"
ssh -i "%USERPROFILE%\.ssh\cicd-demo-dev-deploy" -o IdentitiesOnly=yes root@SERVER_IP
```

The installation uses the existing server password. `exit` returns from SSH to your computer. Later, repeat public-key installation and Docker setup on production; this demo reuses the same key pair.

## 7. Configure GitHub environments and images

**Your computer:** Copy the private key without displaying it. Paste into the environment secret below; keep its contents off-camera.

Mac: `pbcopy < ~/.ssh/cicd-demo-dev-deploy`

Windows CMD: `clip < "%USERPROFILE%\.ssh\cicd-demo-dev-deploy"`

**Each server, through an already verified SSH connection:**

```bash
cat /etc/ssh/ssh_host_ed25519_key.pub
```

Use the first two fields to construct `SERVER_IP ssh-ed25519 BASE64_KEY`. This public host-key entry identifies the server to the runner; it is different from the deployment key you generated.

**GitHub → repo Settings → Environments:** These names match the tutorial's current implementation; ask AI to use them exactly.

| Value | `staging` environment | `production` environment |
| --- | --- | --- |
| Secret: private deployment key | `STAGING_SSH_KEY` | `SSH_KEY` |
| Variable: that server's IP | `STAGING_HOST` | `SERVER_HOST` |
| Variable: login user (`root`) | `STAGING_USER` | `SERVER_USER` |
| Variable: verified host-key line | `STAGING_KNOWN_HOSTS` | `KNOWN_HOSTS` |

For `production`, enable **Required reviewers** and add yourself. Allow self-review for this solo demonstration; disable administrator bypass if you want the gate enforced for your account. Approve from the run's **Review deployments** button after regression passes.

After the first image push: **GitHub package → Package settings → Change visibility → Public**. Repo visibility and package visibility are separate. Public images need no server registry login. If an organization disables Public, its owner must allow public package creation first.

GHCR means **GitHub Container Registry**. Actions supplies `GITHUB_TOKEN` automatically; the publishing job needs `packages: write`. No manually created registry token is needed for this public-image tutorial.

## 8. Commit changes and inspect deployment

**Your computer:** Review the diff and stage only the files you intend to change.

```sh
git status
git diff
git add PATH_TO_CHANGED_FILE
git commit -m "Describe the change"
git push
```

During the tutorial, the initial workflow runs on branch pushes. In the final workflow, open/update a PR targeting `main` for lint/unit/integration checks; merge it for the full release. Branch protection is a separate setting for enforcing passing checks before merge.

**Server — after automated deployment:** The deployment Compose file is created during the tutorial; it is not part of the starter. These inspection commands do not need an `IMAGE_REF` environment variable:

```bash
docker ps
docker logs --tail 50 YOUR_REPO-app-1
docker inspect --format '{{.Config.Image}}' YOUR_REPO-app-1
curl -I http://127.0.0.1:5151/
```

Use the container name printed by `docker ps` if different. The pipeline builds once, deploys the exact image digest to staging, runs browser regression, and promotes that same digest to production after approval. Production gets its own `tasks/` directory; we do not copy staging test data.
