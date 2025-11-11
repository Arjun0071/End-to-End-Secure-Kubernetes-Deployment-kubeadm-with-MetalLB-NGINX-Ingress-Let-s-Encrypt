# 🚀 Secure Kubernetes Deployment (kubeadm) — End-to-End Tutorial
**Kubernetes (kubeadm) + MetalLB + NGINX Ingress + Cert-Manager (Let’s Encrypt)**
  
This README describes a hands-on, end-to-end setup used to deploy a Python web application (packaged via Helm and hosted on Docker Hub) on a small, self-managed Kubernetes cluster. The cluster lives on **private VMs** (Master + Worker) and is exposed securely via a **public Jump Server** using persistent port-forwarding.

> **Public test URL used in this repo (example):** `https://myapi.servebeer.com`  
> **Placeholders you must replace:** `<MASTER_IP>`, `<token>`, `<hash>`, `<your_helm_chart_name>`, `<your_namespace>`, `<your_dockerhub_repo/image:tag>`, `<your_email@example.com>`

---

## Table of Contents
1. [Overview & Architecture] 
2. [Important Note — SSH access]  
3. [Commands for Master & Worker Nodes (common)]  
4. [Master node — initialize control plane] 
5. [Worker node — join the cluster]
6. [Complete Master node — MetalLB, Ingress, Cert-Manager, App Deploy]
7. [Jump Server — kubeconfig, kubectl, persistent port-forwarding] 
8. [Screenshots]  
9. [CI/CD Integration] 

---

## Overview & Architecture

The following diagram explains how external traffic flows securely from the public internet to the private Kubernetes cluster through the Jump Server and Ingress Controller:

![kubeadm-architecture](https://github.com/user-attachments/assets/c65987ea-3479-4ad1-bb70-1e45b8710eea)


Internet  
↓  
DNS → `myapi.servebeer.com`  
↓  
**Jump Server (Public IP)** — persistent `kubectl port-forward` → **Ingress Controller (NGINX)**  
↓  
**Kubernetes Cluster (Private VMs)**  
├─ **Master (Control Plane)** — initialized via `kubeadm`, hosts MetalLB controller, NGINX Ingress, and Cert-Manager  
└─ **Worker Node(s)** — host application pods  

- **MetalLB** allocates `LoadBalancer` IPs from the **private subnet**, simulating cloud-style load balancing for on-prem environments.  
- **Cert-Manager** automates SSL certificate management by obtaining and renewing **Let’s Encrypt** certificates via the **HTTP-01 challenge** handled through the Ingress.

## Important Note — SSH access
Before you start, **enable passwordless SSH from the Jump Server to both private VMs**


## ⚙️ Commands for Master & Worker Nodes (Common)

Run the following commands on **both Master and Worker nodes** (❌ not on the Jump Server unless explicitly mentioned).

---

### 1. 🧩 Update OS Packages

```bash
sudo apt update && sudo apt upgrade -y
```

### 2. 🐋 **Containerd:** [Install & Configure containerd](https://kubernetes.io/docs/tasks/administer-cluster/migrating-from-dockershim/change-runtime-containerd/)
#### Check status
```
sudo systemctl status containerd
```
### ✅ Ensure systemd cgroup is used
```
containerd config dump | grep SystemdCgroup
# Expected: SystemdCgroup = true
```

### If the output is false, edit:
```
sudo vim /etc/containerd/config.toml
# In the section:
# [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
# add/update:
SystemdCgroup = true

sudo systemctl restart containerd
```
### 3. **Kubeadm, Kubelet, Kubectl:** [Install kubeadm, kubelet, kubectl](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
### 4. Disable swap and enable kernel settings
```
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

sudo modprobe br_netfilter

sudo tee /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sudo sysctl --system
```
## Master node — initialize control plane
Run the following on the Master node only.

### 1. Initialize Kubernetes master
(Using Flannel pod network example CIDR)
```
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```
Save the kubeadm output — it contains the join command used by Workers (kubeadm join <MASTER_IP>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>).

### 2. Set up kubectl for your user
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
### 3. Verify API server listening
```
sudo netstat -tulnp | grep 6443   # Kubernetes API server should be listening on 6443
```

### 4. **Flannel CNI:** [Flannel GitHub Repository](https://github.com/flannel-io/flannel)

### 5. Verify Master node state
```
kubectl get nodes
```
# Note: Master may show NotReady until CNI is up


## Worker node — join the cluster
Run on each Worker node using the join command produced by kubeadm init
```
sudo kubeadm join <MASTER_IP>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```
If the token expired, create a new one on the master:
```
# Run on Master
kubeadm token create --print-join-command
```
Verify on the Master:

kubectl get nodes
#### After a few moments, nodes should show as Ready

## Complete Master node — install MetalLB, Ingress, Cert-Manager, and deploy app
After Worker nodes have joined, finish the control-plane configuration and deploy network services & the app.
### 1.  **Helm:** [Install Helm](https://helm.sh/docs/intro/install/)
### 2. **MetalLB:** [Install MetalLB](https://metallb.universe.tf/installation/)
Apply MetalLB address pool configuration (file exists in repo and make changes to it accordingly)
This repo contains metallb-config.yaml — apply that file:

kubectl apply -f metallb-config.yaml

The metallb-config.yaml in this repo defines the IPAddressPool and L2Advertisement used by MetalLB. Edit it if you need a different range for your environment.

### 3.**NGINX Ingress Controller:** [Deploy Ingress NGINX](https://platform9.com/learn/v1.0/tutorials/nginix-controller-via-yaml)
```
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```
The ingress-nginx-controller service should get an EXTERNAL-IP assigned by MetalLB.

### 4. **Cert-Manager:** [Install Cert-Manager](https://cert-manager.io/docs/installation/)
```
kubectl get pods -n cert-manager
```
Apply ClusterIssuer (file exists in repo make changes to it accordingly)
This repo contains clusterissuer.yaml (configures ACME/Let’s Encrypt). Apply it:
```
kubectl apply -f clusterissuer.yaml
```

### 5. Deploy your application using Helm
```
helm install <your_helm_chart_name> ./<path_to_chart> --namespace default --create-namespace
```
Make sure the service exposed by your chart uses the port expected by your ingress (example in your setup: port 8000).

### 6. Apply the Ingress resource (file exists in repo)
This repo contains ingress.yaml (Update the placeholder values accordingly). Apply it:

kubectl apply -f ingress.yaml


ingress.yaml references the ClusterIssuer letsencrypt-prod (cert-manager) and requests a TLS certificate. cert-manager will create a Certificate and attempt ACME HTTP01 validation.

### 7. Verify Cert-Manager & Ingress
```
kubectl get certificate -A
kubectl describe certificate myapi-cert -n default   # or name used in repo
kubectl describe ingress fastapi-ingress -n default  # or the ingress name in repo
```
If Certificate shows Ready=True, the TLS secret exists and you should be able to access the app via HTTPS.


## Jump Server — kubeconfig, kubectl & persistent port-forwarding
Final step: configure the Jump Server so the cluster is accessible externally.

### 1. Copy kubeconfig from Master to Jump Server

On Jump Server:
```
scp <MASTER_USER>@<MASTER_IP>:/etc/kubernetes/admin.conf /root/.kube/config
```
Set proper permissions:
```
sudo mkdir -p /root/.kube
sudo chown root:root /root/.kube/config
sudo chmod 600 /root/.kube/config
ls -l /root/.kube/config
```
### 2. **Kubectl:** [Install kubectl on Jump Server](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)


Verify connectivity:
```
KUBECONFIG=/root/.kube/config kubectl get nodes
# or
kubectl get nodes
```

### 3. Port-forward ingress to public ports 80/443 (quick test)
```
sudo kubectl port-forward svc/ingress-nginx-controller -n ingress-nginx 80:80 443:443 --address 0.0.0.0
```
#### Output will show:
#### Forwarding from 0.0.0.0:80 -> 80
#### Forwarding from 0.0.0.0:443 -> 443

### 4. Make the port-forward persistent using systemd

Create /etc/systemd/system/k8s-portforward.service with the following content (on Jump Server):
```
[Unit]
Description=Kubernetes port-forward for ingress controller (80/443)
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=root
Environment=KUBECONFIG=/root/.kube/config
ExecStart=/usr/local/bin/kubectl port-forward svc/ingress-nginx-controller -n ingress-nginx 80:80 443:443 --address 0.0.0.0
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Enable & start it:
```
sudo systemctl daemon-reload
sudo systemctl enable --now k8s-portforward.service
sudo systemctl status k8s-portforward.service
```
5. Open firewall ports (ufw example (if your ufw is active)
```
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw reload
```
If cert-manager successfully issued the certificate and Ingress is configured properly, https://DNS_HostName should now serve your application with a valid TLS certificate.

## Screenshots:
### Application Accessible via DNS Hostname
<p align="center">
<img width="800" height="400" alt="image" src="https://github.com/user-attachments/assets/e756f246-445d-493e-93fd-e028ac6601e4" />
</p>

### Verified HTTPS Connection
<p align="center">
<img width="800" height="400" alt="image" src="https://github.com/user-attachments/assets/8292b8f0-b444-4e3f-986f-84ff7d83d287" />
</p>













## CI/CD Integration — Automated Build & Deployment via GitHub Actions
This project integrates a fully automated CI/CD pipeline to streamline Docker image creation, Helm upgrades, and cluster deployment on the private Kubernetes setup built using kubeadm.

### Overview

Whenever code is pushed to the main branch, the pipeline automatically:
Builds and pushes the Docker image to Docker Hub.
Triggers deployment on the Kubernetes master node (via Jump Server) using Helm upgrade to roll out the updated image.
This ensures a continuous delivery loop: every code change is built, containerized, and deployed securely to the private cluster.

---

### 🚀 1. Continuous Integration (CI) — Build and Push Docker Image

**Workflow File:** `.github/workflows/ci.yml`

This workflow automates the build and image-push process every time a commit is pushed to the **main branch**.

#### 🔧 Steps

**1. Checkout Repository**  
   Fetches the latest code from the repository.

**2. Docker Hub Authentication**  
   Logs in to Docker Hub using stored **GitHub Secrets**:  
   - `DOCKERHUB_USERNAME`  
   - `DOCKERHUB_TOKEN`

**3. Build Docker Image**  
   Builds a Docker image of the FastAPI application and tags it with the unique Git commit SHA for version traceability:
   ```bash
   docker build -t <DOCKERHUB_USERNAME>/fastapi-demo:<GITHUB_SHA> .
   ```
**4. Push Image to Docker Hub**
   Pushes the built image to Docker Hub:
   ```
   docker push <DOCKERHUB_USERNAME>/fastapi-demo:<GITHUB_SHA>
   ```

**5. Expose Image Tag for CD Pipeline**
   Exports the built image tag (GITHUB_SHA) so the CD workflow can use it during deployment.

## 🧩 2. Continuous Deployment (CD) — Helm-Based Deployment via Jump Server

**Workflow File:** `.github/workflows/cd.yml`

This workflow triggers automatically after the **CI workflow** completes successfully.  
It securely connects to the **Jump Server**, which has SSH access to the **Kubernetes master node**, and deploys the latest Docker image using **Helm**.

---

### 🔧 Steps

**1. Checkout Repository**  
   Clones the repository to access the Helm chart.

**2. Set Up SSH Authentication**  
   Configures the SSH agent using a private key stored in **GitHub Secrets**:  
   - `SSH_PRIVATE_KEY`

**3. Deploy via Jump Server**  
   Establishes a secure SSH connection to the Jump Server, then connects to the Kubernetes master node to perform the Helm upgrade.  

   Uses these environment variables (defined as GitHub Secrets):  
   - `JUMP_USER`, `JUMP_HOST` — Jump Server credentials  
   - `MASTER_USER`, `MASTER_IP` — Master node credentials  
   - `DOCKERHUB_USERNAME` — Docker Hub username  

**4. Helm Upgrade Execution**  
   On the master node, the workflow ensures the Helm chart exists at:  
   `/home/<MASTER_USER>/FastAPI-CI-CD/demoapp-chart`  

   Then executes:
   ```
   helm upgrade --install fastapi-release /home/<MASTER_USER>/FastAPI-CI-CD/demoapp-chart \
     --reuse-values \
     --set image.repository=<DOCKERHUB_USERNAME>/fastapi-demo \
     --set image.tag=<GITHUB_SHA> \
     --set image.pullPolicy=Always \
     --namespace default \
     --atomic
   ```
  
  The --reuse-values flag ensures existing service configurations (like NodePort) are preserved while updating only the image

  ## 🔐 3. Secrets Used

| Secret Name | Description |
|--------------|-------------|
| `DOCKERHUB_USERNAME` | Docker Hub username |
| `DOCKERHUB_TOKEN` | Docker Hub access token |
| `SSH_PRIVATE_KEY` | Private key for SSH access |
| `JUMP_USER`, `JUMP_HOST` | Jump Server SSH credentials |
| `MASTER_USER`, `MASTER_IP` | Kubernetes Master Node credentials |

---

## 🧠 4. Workflow Trigger

The **CD workflow** runs only after the **CI workflow** (`CI- Build and push the Docker Image`) completes successfully:

```yaml
on:
  workflow_run:
    workflows: ["CI- Build and push the Docker Image"]
    types:
      - completed
```
This ensures only a successfully built and pushed image is deployed to the cluster.

## 📊 5. Summary

| Stage | Tool | Description |
|--------|------|-------------|
| **CI** | Docker + GitHub Actions | Builds and pushes the FastAPI image to Docker Hub |
| **CD** | Helm + SSH + Jump Server | Securely deploys the updated image to the Kubernetes cluster |
| **Security** | GitHub Secrets | Manages all sensitive credentials safely |

---

## End Result

Each new commit on the **main branch** automatically:  
- Builds a Docker image  
- Pushes it to **Docker Hub**  
- Triggers a **Helm-based deployment** to your private Kubernetes cluster  

> Achieving a **zero-touch, fully automated CI/CD pipeline** — secure, version-controlled, and production-ready.
