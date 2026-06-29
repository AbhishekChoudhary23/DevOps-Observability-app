# Enterprise Kubernetes GitOps Platform

An end-to-end **GitOps-based Kubernetes CI/CD pipeline** that demonstrates enterprise deployment practices using **Terraform**, **Jenkins**, **ArgoCD**, **Docker Hub**, **Prometheus**, **Grafana**, and **NGINX Ingress** on a local **Minikube** cluster.

The project follows a GitOps workflow where infrastructure is provisioned using Terraform, Jenkins builds and publishes container images, ArgoCD continuously deploys Kubernetes manifests, and Prometheus/Grafana provide complete observability.

---

# Architecture

```text
                        Windows Host
                 ┌────────────────────────┐
                 │ Developer Environment  │
                 │ Browser                │
                 │ hosts file             │
                 └───────────┬────────────┘
                             │
                             ▼
                  Kali Linux Virtual Machine
          ┌─────────────────────────────────────────┐
          │            Minikube Cluster             │
          │                                         │
          │ Terraform                               │
          │      │                                  │
          │      ▼                                  │
          │ Kubernetes Infrastructure               │
          └─────────────────────────────────────────┘
                             │
        ┌────────────────────┴────────────────────┐
        │                                         │
        ▼                                         ▼
  GitHub Actions                            Jenkins
   (dev branch)                          (prod branch)
        │                                         │
        └───────────────┬─────────────────────────┘
                        ▼
                  Docker Hub Images
                        │
                        ▼
                GitOps Manifest Repo
                        │
                        ▼
                     ArgoCD
                        │
                        ▼
               Kubernetes Deployments
                        │
        ┌───────────────┴───────────────┐
        ▼                               ▼
    Frontend                        Backend
                                        │
                                        ▼
                            Prometheus ServiceMonitor
                                        │
                                        ▼
                                   Prometheus
                                        │
                                        ▼
                                    Grafana
```

---

# Technology Stack

| Category               | Technology     |
| ---------------------- | -------------- |
| Infrastructure as Code | Terraform      |
| CI (Development)       | GitHub Actions |
| CI (Production)        | Jenkins        |
| CD                     | ArgoCD         |
| GitOps                 | App-of-Apps    |
| Kubernetes             | Minikube       |
| Registry               | Docker Hub     |
| Monitoring             | Prometheus     |
| Dashboards             | Grafana        |
| Ingress                | NGINX Ingress  |


---

# Prerequisites

Install the following tools before starting:

- Docker
- Minikube
- kubectl
- Terraform
- Helm
- Git

---

# 1. Infrastructure Provisioning

The Kubernetes infrastructure for this project is provisioned using **Terraform**.

The Terraform configuration provisions the foundational Kubernetes resources required by the platform, including namespaces and other infrastructure components used by Jenkins, ArgoCD, Prometheus, Grafana, and the application workloads.

The Terraform code is available in the repository under the **`terraform/`** directory.

```text
terraform/
├── provider.tf
├── main.tf
└── ...
```

Initialize and apply the Terraform configuration:

```bash
cd terraform

terraform init
terraform plan
terraform apply
```

The Terraform configuration is available in the [terraform](https://github.com/AbhishekChoudhary23/DevOps-Observability-k8s-gitops-manifest/tree/main/terraform) directory.

---

# 2. Start the Kubernetes Cluster

Start Minikube:

```bash
minikube start
```

Enable the NGINX Ingress Controller:

```bash
minikube addons enable ingress
```

---

# 3. Deploy CI/CD Platform

## 3.1 Jenkins

Deploy Jenkins into the `jenkins` namespace.

Install the following Jenkins plugins:

- Kubernetes Plugin
- Docker Pipeline Plugin

Configure the following credentials:

- `dockerhub-creds`
- `github-gitops-creds`

---

## 3.2 ArgoCD

Deploy ArgoCD into the `argocd` namespace.

Bootstrap the GitOps repository using the root application:

```bash
kubectl apply -f https://raw.githubusercontent.com/AbhishekChoudhary23/DevOps-Observability-k8s-gitops-manifest/main/argocd-root/my-app-metadata.yaml
```

The project follows the **App-of-Apps** pattern so ArgoCD recursively manages all applications contained within the GitOps repository.

---

# 4. Continuous Integration
## Development Pipeline (GitHub Actions)

The dev branch is integrated with GitHub Actions, providing lightweight CI for development changes.

The workflow:

1. Triggers on every push to the dev branch.
2. Builds the frontend and backend Docker images.
3. Pushes development images to Docker Hub using development tags.
4. Performs automated validation before code reaches production.

This enables rapid feedback for development without requiring the Jenkins infrastructure.

## Production Pipeline (Jenkins)

The prod branch is managed by Jenkins running inside Kubernetes.

The Jenkins pipeline:

1. Polls the production branch every two minutes.
2. Launches a dynamic Kubernetes agent.
3. Builds production Docker images.
4. Pushes versioned images to Docker Hub.
5. Clones the GitOps repository.
6. Updates Kubernetes manifests with the new image tags.
7. Commits and pushes the updated manifests.

Example:

```text
abhishekdocker03/gitops-backend:prod-15
```
---

## Continuous Deployment (ArgoCD)

ArgoCD continuously monitors the GitOps repository.

Whenever Jenkins updates the deployment manifests:

- ArgoCD detects the new Git commit.
- Synchronizes Kubernetes manifests.
- Deploys the latest images.
- Automatically reconciles any configuration drift.
- Restores deleted resources through self-healing.

If any managed Kubernetes resource is manually modified or deleted, ArgoCD restores the desired state.

---

# 5. Observability

The project uses the **kube-prometheus-stack** Helm chart for monitoring.

Components include:

- Prometheus
- Grafana
- Alertmanager
- Node Exporter
- kube-state-metrics

---

## Prometheus Service Discovery

The backend application exposes Prometheus metrics at `/metrics`.

A `ServiceMonitor` enables automatic service discovery.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor

metadata:
  name: app-service-monitor
  namespace: monitoring
  labels:
    release: kube-prometheus-stack

spec:
  namespaceSelector:
    matchNames:
      - my-gitops-app

  selector:
    matchLabels:
      app: backend

  endpoints:
    - port: web
      path: /metrics
```

---

# 6. Networking

## Internal Routing

The React frontend uses NGINX as a reverse proxy to communicate with the backend service through Kubernetes DNS.

```nginx
location /api/ {
    proxy_pass http://backend-service:5000;
}
```

This approach avoids CORS issues while keeping all internal communication within the Kubernetes cluster.

---

## External Access

Traffic flows through the NGINX Ingress Controller.

```text
my-observability-app.local
           │
           ▼
NGINX Ingress Controller
           │
           ▼
Frontend Service
           │
           ▼
Backend Service
```

---

## Windows Host Configuration

Add the Kali VM IP address to the Windows hosts file.

```
C:\Windows\System32\drivers\etc\hosts
```

Example:

```text
192.168.x.x    my-observability-app.local
```

---

# 7. Development Automation

The repository includes a helper script named **`start-devops.sh`** that automates the local development environment.

The script performs the following tasks:

- Starts Minikube if it is not running.
- Enables the NGINX Ingress addon.
- Cleans up stale port-forward processes.
- Creates background port-forwards for:
  - NGINX Ingress
  - Jenkins
  - ArgoCD
  - Prometheus
  - Grafana

Make the script executable:

```bash
chmod +x start-devops.sh
```

Run the script:

```bash
./start-devops.sh
```

---

# Access URLs

| Service | URL |
|----------|-----|
| Application | http://my-observability-app.local |
| ArgoCD | http://my-observability-app.local:8080 |
| Jenkins | http://my-observability-app.local:8081 |
| Prometheus | http://my-observability-app.local:9090 |
| Grafana | http://my-observability-app.local:8082 |

---

# Project Highlights

- Infrastructure provisioned using Terraform
- Dual CI pipeline implementation
  - GitHub Actions for Development
  - Jenkins for Production
- GitOps deployment using ArgoCD
- App-of-Apps architecture
- Dynamic Jenkins Kubernetes agents
- Automated Docker image versioning
- Automatic GitOps manifest updates
- Self-healing Kubernetes deployments
- Prometheus monitoring with ServiceMonitor
- Grafana dashboards
- NGINX Ingress routing
- Enterprise-style local Kubernetes platform

---

# Future Improvements

- GitHub Webhooks (replace SCM polling)
- SonarQube code quality analysis
- Trivy image vulnerability scanning
- Argo Rollouts (Blue-Green / Canary Deployments)
- External Secrets Operator
- HashiCorp Vault integration
- Loki + Promtail centralized logging
- Alertmanager email/Slack notifications
- Horizontal Pod Autoscaler (HPA)