# CSD07 — Portfolio Website + Kubernetes Deployment

A static portfolio website (`index.html`) paired with Kubernetes manifests
to deploy a Flask backend app. Built as part of the Generation Singapore
Cloud & DevOps programme.

---

## ELI5 — Explain Like I'm 5

| Term | ELI5 |
| --- | --- |
| **HTML file** | A document your browser reads and turns into a webpage |
| **Static site** | A website with no server — just files the browser opens directly |
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
├── index.html          # Static portfolio website — open directly in browser
├── static/
│   └── photo.jpg       # Profile photo used in the hero section
└── k8s/
    ├── deployment.yaml # Flask app Kubernetes Deployment
    ├── service.yaml    # LoadBalancer Service (port 5000)
    ├── postgres.yaml   # PostgreSQL Deployment + PVC + ClusterIP Service
    ├── flask.yaml      # (duplicate of deployment.yaml — do not apply both)
    └── secret.yaml     # DB credentials — gitignored, never commit real values
```

---

## Part 1 — Static Portfolio Website

`index.html` is a self-contained static site. No server or install needed.

### How to open it

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
| Projects | 4 project cards with source and live app links |
| Footer | Languages used, contributors |

### Live app links in the portfolio

| Project | URL |
| --- | --- |
| Flask Portfolio | http://40.90.189.161:5000 |
| Flask E-Commerce | http://40.90.189.161:5000/ecommerce |
| Flask Blog | http://40.90.189.161:5000/flaskwebsite |

> These URLs point to the AKS (Azure) public IP — accessible from anywhere.
> `localhost:5000` is the local Docker Desktop cluster — only works on your PC.

---

## Part 2 — Kubernetes Architecture

```
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
- On Docker Desktop: accessible at `localhost:5000`
- On AKS: accessible at the public IP (`40.90.189.161:5000`)

#### `k8s/postgres.yaml` — PostgreSQL
- Runs `postgres:15-alpine`
- `PGDATA=/var/lib/postgresql/data/pgdata` — prevents conflict with PVC mount
- PVC: 1Gi persistent storage so data survives pod restarts
- Service: `ClusterIP` — only reachable inside the cluster (Flask connects to it as `postgres-service:5432`)

#### `k8s/secret.yaml` — Credentials (DO NOT COMMIT)
- Stores `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB`, `DATABASE_URL`
- This file is gitignored — never commit real passwords to git
- Create the secret manually instead (see setup below)

---

## How to Deploy Locally (Docker Desktop)

### Prerequisites
- Docker Desktop installed with Kubernetes enabled
- `kubectl` available in terminal

### Step 1 — Switch to Docker Desktop cluster

```powershell
kubectl config use-context docker-desktop
```

### Step 2 — Create secrets (first time only)

```powershell
# DB credentials
kubectl create secret generic flask-db-secret `
  --from-literal=POSTGRES_USER=flaskuser `
  --from-literal=POSTGRES_PASSWORD=<your-password> `
  --from-literal=POSTGRES_DB=flaskapp `
  "--from-literal=DATABASE_URL=postgresql://flaskuser:<your-password>@postgres-service:5432/flaskapp"

# Docker Hub pull secret (needed because image is private)
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

> Do NOT apply `flask.yaml` — it's a duplicate of `deployment.yaml` and will conflict.

### Step 4 — Verify

```powershell
kubectl get pods      # both pods should show Running
kubectl get services  # flask-service should show EXTERNAL-IP: localhost
```

### Step 5 — Open the app

Go to **http://localhost:5000** in your browser.

---

## Two Environments

| | Local (Docker Desktop) | Cloud (AKS) |
| --- | --- | --- |
| URL | `http://localhost:5000` | `http://40.90.189.161:5000` |
| Who can access | Only your PC | Anyone on the internet |
| Where it runs | Docker Desktop K8s | Azure Kubernetes Service |
| Switch context | `kubectl config use-context docker-desktop` | `kubectl config use-context flask-portfolio-aks` |

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

- Static HTML sites need no server — just open in browser
- Kubernetes needs secrets created before pods can start
- Never commit `secret.yaml` with real passwords — create secrets via `kubectl` command
- `DATABASE_URL` hostname must match the K8s service name (`postgres-service`), not the Docker Compose name (`db`)
- `ClusterIP` is for internal traffic only; `LoadBalancer` exposes to outside
- `PGDATA` must point to a subdirectory to avoid the `lost+found` conflict on PVC mounts
- `imagePullSecrets` is required when pulling from a private Docker Hub repo
- Do not apply duplicate manifests with the same resource name — last one wins and may cause confusion
- Docker Desktop K8s shares the Docker engine — lighter than minikube for local dev
