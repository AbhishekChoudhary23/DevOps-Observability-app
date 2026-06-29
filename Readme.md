
### 📄 README.md
```markdown
# Enterprise GitOps & Observability Pipeline

This repository contains the infrastructure and GitOps configurations for a complete, self-healing Kubernetes CI/CD pipeline. The architecture bridges a Windows development host with a Kali Linux virtual machine running a Minikube cluster.

## 🏛️ Architecture Overview

* **Developer Environment:** Windows (Host) / Kali Linux (VM)
* **Kubernetes Engine:** Minikube
* **CI Pipeline:** Jenkins (Dynamic Docker-in-Docker Pod Agents)
* **CD Pipeline:** ArgoCD (App-of-Apps Pattern)
* **Container Registry:** Docker Hub
* **Observability:** Prometheus & Grafana (`kube-prometheus-stack`)
* **Ingress Routing:** Nginx Ingress Controller

---

## 1. Initial Infrastructure Setup (Kali VM)

Since this project runs locally, we bypass Terraform and use Minikube to provision the Kubernetes node directly.

### 1.1 Start the Cluster & Enable Addons
```bash
# Start the Minikube cluster
minikube start

# Enable the Nginx Ingress Controller
minikube addons enable ingress

```
### 1.2 Create Logical Namespaces
```bash
kubectl create namespace jenkins
kubectl create namespace argocd
kubectl create namespace monitoring
kubectl create namespace my-gitops-app

```
## 2. CI/CD Tooling Deployment
### 2.1 Deploy Jenkins
Deployed Jenkins to the jenkins namespace and installed the **Docker Pipeline** and **Kubernetes** plugins. Created the following credentials in the Jenkins UI:
 * dockerhub-creds
 * github-gitops-creds
### 2.2 Deploy ArgoCD
Deployed ArgoCD to the argocd namespace. Configured the "App-of-Apps" root metadata file to recursively track our GitOps repository.
**Command Executed:**
```bash
kubectl apply -f https://raw.githubusercontent.com/AbhishekChoudhary23/DevOps-Observability-k8s-gitops-manifest/main/argocd-root/my-app-metadata.yaml

```
## 3. The GitOps Workflow
### 3.1 Continuous Integration (Jenkinsfile)
The application repository contains a Jenkinsfile that utilizes a dynamic Kubernetes agent to mount the VM's Docker socket.
 1. Polls GitHub every 2 minutes.
 2. Compiles Frontend and Backend images.
 3. Pushes to Docker Hub (abhishekdocker03/gitops-backend:prod-X).
 4. Clones this GitOps repository, updates the YAML manifest image tags via sed, and pushes the changes back to GitHub.
### 3.2 Continuous Deployment (ArgoCD)
ArgoCD monitors the root directory with recurse: true. When Jenkins pushes new tags to the manifest repository, ArgoCD automatically:
 1. Syncs the Frontend and Backend Deployments inside my-gitops-app.
 2. Restarts the Pods with the new Docker Hub images.
 3. Maintains self-healing (if a deployment is manually deleted, ArgoCD restores it).

## 4. Observability Setup
Deployed the kube-prometheus-stack into the monitoring namespace.
### 4.1 Prometheus Service Discovery
We configured the Backend Service to expose a named port (web) and a specific label (release: kube-prometheus-stack). We then deployed a ServiceMonitor to tell Prometheus to scrape this endpoint automatically:
```yaml
# monitoring/service-monitor.yaml
apiVersion: [monitoring.coreos.com/v1](https://monitoring.coreos.com/v1)
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
## 5. Networking & Ingress
### 5.1 Internal Routing (Nginx)
The Frontend React app uses Nginx to proxy API calls internally using Kubernetes DNS, bypassing CORS and local network isolation:
```nginx
location /api/ {
    proxy_pass http://backend-service:5000;
}

```
### 5.2 External Ingress
We deployed an Ingress resource to route traffic from my-observability-app.local to the frontend-service on port 80.
### 5.3 Local DNS Resolution (Windows Host)
Added the Kali VM's IP address to the Windows C:\Windows\System32\drivers\etc\hosts file:
```text
<KALI_VM_IP>    my-observability-app.local

```
```

---

### `start-devops.sh` (The Automation Script)

Create this file on your **Kali VM**. It will start your cluster (if it's off), ensure ingress is enabled, kill any old stuck tunnels, and launch all your port-forwards into detached background processes using `nohup`.

Create the file:
```bash
nano start-devops.sh

```
Paste this code:
```bash
#!/bin/bash

echo "Starting Enterprise DevSecOps Environment..."

# 1. Ensure Minikube is running
echo "Checking Minikube status..."
minikube status | grep -q "Running" || minikube start

# 2. Ensure Ingress Addon is enabled
echo "Verifying Ingress controller..."
minikube addons enable ingress > /dev/null 2>&1

# 3. Clean up old port-forwards to prevent port conflicts
echo "Cleaning up old network tunnels..."
sudo pkill -f "port-forward"

echo "Establishing detached background tunnels (Binding to 0.0.0.0)..."

# 4. Route Application Ingress (Requires sudo to bind to port 80)
# We use sudo -E to preserve the user's ~/.kube/config path
echo "  -> Starting Nginx Ingress on Port 80..."
sudo -E nohup kubectl port-forward svc/ingress-nginx-controller -n ingress-nginx 80:80 --address 0.0.0.0 > /tmp/ingress-pf.log 2>&1 &

# 5. Route Management Control Planes
echo "  -> Starting ArgoCD on Port 8080..."
nohup kubectl port-forward svc/argo-cd-argocd-server -n argocd 8080:443 --address 0.0.0.0 > /tmp/argocd-pf.log 2>&1 &

echo "  -> Starting Jenkins on Port 8081..."
nohup kubectl port-forward svc/jenkins -n jenkins 8081:8080 --address 0.0.0.0 > /tmp/jenkins-pf.log 2>&1 &

echo "  -> Starting Prometheus on Port 9090..."
nohup kubectl port-forward svc/kube-prometheus-stack-prometheus -n monitoring 9090:9090 --address 0.0.0.0 > /tmp/prom-pf.log 2>&1 &

echo "  -> Starting Grafana on Port 8082..."
nohup kubectl port-forward svc/kube-prometheus-stack-grafana -n monitoring 8082:80 --address 0.0.0.0 > /tmp/grafana-pf.log 2>&1 &

echo ""
echo "All systems go! Tunnels are running in the background."
echo "--------------------------------------------------------"
echo "Live App: http://my-observability-app.local"
echo "ArgoCD:   http://my-observability-app.local:8080"
echo "Jenkins:  http://my-observability-app.local:8081"
echo "Prom:     http://my-observability-app.local:9090"
echo "Grafana:  http://my-observability-app.local:8082"
echo "--------------------------------------------------------"
echo "(To stop the tunnels later, just run: sudo pkill -f port-forward)"

```
Make the script executable:
```bash
chmod +x start-devops.sh

```