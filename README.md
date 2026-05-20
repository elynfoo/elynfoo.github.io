# CSD07 — Portfolio Website + Kubernetes Deployment

A static portfolio website (`index.html`) paired with Kubernetes manifests
to deploy a Flask backend app. Built as part of the Generation Singapore
Cloud & DevOps programme. Live at [elynfoo.github.io](https://elynfoo.github.io).

---

## ELI5 — Explain Like I'm 5

| Term | ELI5 |
| --- | --- |
| **HTML file** | A document your browser reads and turns into a webpage |
| **Static site** | A website with no server — just files the browser opens directly |
| **GitHub Pages** | GitHub's free hosting for static sites — reads your HTML and serves it live |
| **GitHub Actions** | An automated worker that runs tasks (like deploying) every time you push code |
| **Kubernetes** | A manager that runs and watches your app containers — restarts them if they crash |
| **Pod** | One running container inside Kubernetes |
| **Deployment** | Tells Kubernetes how many pods to run and which image to use |
| **Service** | The door/address to reach your pod from outside |
| **Secret** | A locked safe in Kubernetes that stores passwords — never hardcoded in files |
| **PVC** | A USB drive that keeps database data even if the pod restarts |
| **ClusterIP** | A service only reachable inside the cluster (used for Postgres) |
| **LoadBalancer** | A service reachable from outside the cluster (used for Flask) |

---

## Project Structure

```
CSD07/
├── index.html                      # Static portfolio website
├── static/
│   └── photo.jpg                   # Profile photo used in the hero section
├── .github/
│   └── workflows/
│       └── deploy.yml              # GitHub Actions — auto deploys to GitHub Pages
└── k8s/
    ├── deployment.yaml             # Flask app Kubernetes Deployment
    ├── service.yaml                # LoadBalancer Service (port 5000)
    ├── postgres.yaml               # PostgreSQL Deployment + PVC + ClusterIP Service
    ├── flask.yaml                  # (duplicate of deployment.yaml — do not apply both)
    └── secret.yaml                 # DB credentials — gitignored, never commit real values
```

---

## Part 1 — Static Portfolio Website

`index.html` is a self-contained static site. No server or install needed.

### Live Site

[https://elynfoo.github.io](https://elynfoo.github.io)

Deployed automatically via GitHub Actions on every push to `main`.

### How to open locally

Double-click `index.html` in File Explorer, or run in PowerShell:

```powershell
Start-Process "c:\AzureDevops\CSD07\index.html"
```

### What's inside

| Section | Description |
| --- | --- |
| Hero | Name, title, email, LinkedIn link |
| About Me | Background summary |
| Digital Presence | Links to GitHub, Docker Hub, Azure DevOps |
| Experience | Generation SG bootcamp + personal projects |
| Skills | Cloud & DevOps, Backend, Infrastructure, Soft Skills |
| Projects | 5 project cards with source and live app links |
| Architecture | GitHub Actions deploy workflow |
| Footer | Languages used, contributors |

### Projects featured

| Project | Tech Stack | Live URL |
| --- | --- | --- |
| Containerized Flask Portfolio | Python, Docker, Kubernetes, PostgreSQL | [http://40.90.189.161:5000](http://40.90.189.161:5000) |
| Flask E-Commerce App | Python, Flask, PostgreSQL | [http://40.90.189.161:5000/ecommerce](http://40.90.189.161:5000/ecommerce) |
| Flask Comment Blog | Python, Flask-Login, PostgreSQL | [http://40.90.189.161:5000/flaskwebsite](http://40.90.189.161:5000/flaskwebsite) |
| Terraform + Ansible VM | Terraform, Ansible, Azure, Python | [http://40.90.189.161:5001](http://40.90.189.161:5001) |
| Web Network Scanner | Bash, PHP, Apache, Nmap, Linux | Local only |

> Live URLs point to the AKS public IP — accessible from anywhere.

---

## Part 2 — GitHub Actions CI/CD

The portfolio deploys automatically via `.github/workflows/deploy.yml`.

```
git push github main
        ↓
GitHub Actions triggers
        ↓
Checkout → Configure Pages → Upload artifact → Deploy
        ↓
Live at elynfoo.github.io
```

### Remote setup

| Remote | URL | Purpose |
| --- | --- | --- |
| `github` | github.com/elynfoo/elynfoo.github.io | Only remote — source + Pages deploy |

To push changes:

```powershell
git push github main
```

---

## Part 3 — Kubernetes Architecture

```text
[ Browser ]
     |
     | HTTP :5000
     v
[ flask-service ]          <- LoadBalancer (external access)
     |
     v
[ flask-portfolio Pod ]    <- Deployment, image: elynfoo/devops-flaskapp:latest
     |                        pulls DATABASE_URL from flask-db-secret
     | PostgreSQL :5432
     v
[ postgres-service ]       <- ClusterIP (internal only)
     |
     v
[ postgres Pod ]           <- Deployment, image: postgres:15-alpine
     |                        pulls POSTGRES_USER/PASSWORD/DB from flask-db-secret
     v
[ postgres-pvc ]           <- PersistentVolumeClaim (1Gi storage)
```

### Kubernetes files explained

#### `k8s/deployment.yaml` — Flask App

- Runs 1 replica of `elynfoo/devops-flaskapp:latest`
- `imagePullPolicy: Always` — always pulls latest image from Docker Hub
- `imagePullSecrets: dockerhub-secret` — needed because Docker Hub repo is private
- Reads `DATABASE_URL` from `flask-db-secret`

#### `k8s/service.yaml` — Flask Service

- Type: `LoadBalancer` — exposes the app on port 5000
- On AKS: accessible at the public IP (`40.90.189.161:5000`)

#### `k8s/postgres.yaml` — PostgreSQL

- Runs `postgres:15-alpine`
- `PGDATA=/var/lib/postgresql/data/pgdata` — prevents conflict with PVC mount
- PVC: 1Gi persistent storage so data survives pod restarts
- Service: `ClusterIP` — only reachable inside the cluster

#### `k8s/secret.yaml` — Credentials (DO NOT COMMIT)

- Stores `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB`, `DATABASE_URL`
- This file is gitignored — never commit real passwords to git
- Create the secret manually instead (see setup below)

---

## How to Deploy to AKS

### Prerequisites

- Azure CLI installed and logged in
- `kubectl` available in terminal
- AKS credentials configured

### Step 1 — Connect to AKS

```powershell
az aks get-credentials --resource-group flask-portfolio-rg --name flask-portfolio-aks
```

### Step 2 — Create secrets (first time only)

```powershell
# DB credentials
kubectl create secret generic flask-db-secret `
  --from-literal=POSTGRES_USER=flaskuser `
  --from-literal=POSTGRES_PASSWORD=<your-password> `
  --from-literal=POSTGRES_DB=flaskapp `
  "--from-literal=DATABASE_URL=postgresql://flaskuser:<your-password>@postgres-service:5432/flaskapp"

# Docker Hub pull secret
kubectl create secret docker-registry dockerhub-secret `
  --docker-server=https://index.docker.io/v1/ `
  --docker-username=elynfoo `
  --docker-password=<access-token> `
  --docker-email=elynf@genstudents.org
```

### Step 3 — Apply manifests

```powershell
kubectl apply -f k8s/postgres.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
```

> Do NOT apply `flask.yaml` — it is a duplicate of `deployment.yaml` and will conflict.

### Step 4 — Verify

```powershell
kubectl get pods      # both pods should show Running
kubectl get services  # flask-service should show EXTERNAL-IP
```

---

## Common kubectl Commands

```powershell
# Check what's running
kubectl get pods
kubectl get services
kubectl get secrets

# View logs if a pod is crashing
kubectl logs deployment/flask-portfolio

# Restart the Flask app
kubectl rollout restart deployment/flask-portfolio

# Delete everything and start fresh
kubectl delete -f k8s/postgres.yaml
kubectl delete -f k8s/deployment.yaml
kubectl delete -f k8s/service.yaml
```

---

## Key Learnings

- Static HTML sites need no server — just open in browser or push to GitHub Pages
- GitHub Actions deploys to GitHub Pages automatically on every push to `main`
- Kubernetes needs secrets created before pods can start
- Never commit `secret.yaml` with real passwords — create secrets via `kubectl` command
- `DATABASE_URL` hostname must match the K8s service name (`postgres-service`), not the Docker Compose name (`db`)
- `ClusterIP` is for internal traffic only; `LoadBalancer` exposes to outside
- `PGDATA` must point to a subdirectory to avoid the `lost+found` conflict on PVC mounts
- `imagePullSecrets` is required when pulling from a private Docker Hub repo
- Do not apply duplicate manifests with the same resource name — last one wins and may cause confusion
