# DevOps Portfolio — CI/CD Pipeline & Infrastructure as Code

## Introduction

This project demonstrates the principles of continuous integration, continuous delivery, and infrastructure as code using a containerised Flask web application.

The pipeline automates every stage from code commit to live deployment — building a Docker image, pushing it to Docker Hub, and applying Kubernetes manifests — without any manual steps.

---

## Requirements

**Accounts**
- [Azure DevOps](https://dev.azure.com) account with a project and repo
- [Docker Hub](https://hub.docker.com) account
- [Microsoft Azure](https://portal.azure.com) subscription (for Terraform/Ansible VM deployment)

**Tools installed locally**
- Docker Desktop
- kubectl
- Minikube
- Terraform >= 1.0
- Ansible >= 2.9
- Python 3.x + pip
- Azure CLI (`az`)

---

## Environment Variables & Pipeline Secrets

Pipeline variables must be set in Azure DevOps — **never hardcode credentials in code**.

Navigate to: **Pipeline → Edit → Variables**

| Variable | Description |
|----------|-------------|
| `DOCKER_USERNAME` | Docker Hub username |
| `DOCKER_PASSWORD` | Docker Hub password or access token |
| `AZURE_SUBSCRIPTION_ID` | Azure subscription ID (for Terraform) |
| `AZURE_CLIENT_ID` | Service principal app ID |
| `AZURE_CLIENT_SECRET` | Service principal secret |
| `AZURE_TENANT_ID` | Azure tenant ID |

To verify locally:
```bash
echo $DOCKER_USERNAME
echo $AZURE_SUBSCRIPTION_ID
az account show
```

---

## Pipeline Overview

The `azure-pipelines.yml` defines the full CI/CD flow triggered on every push to `main`:

```
Commit → Azure Repos
       → Azure DevOps trigger fires
       → Docker image built
       → Image pushed to Docker Hub
       → Kubernetes manifest applied
       → App live
```

15 manual steps reduced to zero.

---

## Docker Build & Push

Build and test the image locally before the pipeline:

```bash
# Build
docker build -t <dockerhub-username>/flask-app:latest .

# Run locally
docker run -p 5000:5000 <dockerhub-username>/flask-app:latest

# Push
docker push <dockerhub-username>/flask-app:latest
```

Verify the image appears on [hub.docker.com](https://hub.docker.com).

---

## Kubernetes Deployment (Minikube)

Start Minikube and apply the manifests:

```bash
minikube start

kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml

# Verify pods are running
kubectl get pods
kubectl get services
```

Expected output:
```
NAME                        READY   STATUS    RESTARTS
flask-app-xxxxxxxxx-xxxxx   1/1     Running   0
```

Access the app:
```bash
minikube service flask-app-service --url
```

---

## Infrastructure as Code (Terraform + Ansible)

### Terraform — Provision the Azure VM

```bash
cd infra/terraform

terraform init
terraform plan
terraform apply
```

Terraform outputs the VM's public IP address automatically.

### Ansible — Configure the VM

Ansible picks up the IP from Terraform output and configures the server:

```bash
cd infra/ansible

ansible-playbook -i inventory.ini playbook.yml
```

The playbook:
1. Installs Python and Flask
2. Copies the application files
3. Starts the Flask service

Destroy and re-provision at any time:
```bash
terraform destroy
terraform apply
```

---

## Final Result

If the pipeline runs successfully you should see:

- ✅ Azure DevOps pipeline shows all stages green
- ✅ Docker Hub shows the latest image tag
- ✅ `kubectl get pods` shows `Running` status
- ✅ Flask app accessible via Minikube service URL or Azure VM public IP

**Live demo:** [http://40.90.189.161:5000/ecommerce](http://40.90.189.161:5000/ecommerce)

---

## Project Structure

```
.
├── app/                  # Flask application
├── templates/            # Jinja2 HTML templates
├── static/               # CSS, JS, images
├── k8s/
│   ├── deployment.yaml
│   └── service.yaml
├── infra/
│   ├── terraform/        # Azure VM provisioning
│   └── ansible/          # Server configuration
├── Dockerfile
├── docker-compose.yml
├── azure-pipelines.yml
└── requirements.txt
```

---

## Author

**Elyn Foo** — Junior Cloud & DevOps Engineer, Singapore  
[GitHub](https://github.com/elynfoo) · [LinkedIn](https://www.linkedin.com/in/yi-ling-foo-659b2a143) · [Docker Hub](https://hub.docker.com/u/elynfoo)
