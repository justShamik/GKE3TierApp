🧩 GKE 3-Tier Sample Application
This repository contains a sample 3-tier microservice application (TO-DO List) designed to run on the GKE 3-tier Terraform infrastructure (see GKE3TierTerra). It showcases the integration of a frontend, backend API, and database running across public and private subnets.

📦 Components
Frontend: A simple web UI exposed externally via LoadBalancer.

Backend (API): A RESTful service that connects to the database.

Database: A managed database private to the VPC.

🔗 Architecture Overview

[Internet] → LoadBalancer → Frontend Pod(s) → Backend Pod(s) → Database (private subnet)
                                             
⚙️ Prerequisites
Before deploying this app, you need:

A working instance of GKE3TierTerra deployed on GCP

Jenkins running on a seperate VM with connectivity to the VPC

Ensure the following tools and permissions are set up.

### 🔧 Pre-requisite Setup Script

Run this on your bastion VM or admin machine connected to GCP:

```bash
#!/bin/bash

set -e

# Variables (update these before running)
PROJECT_ID="<YOUR_PROJECT_NAME>"
CLUSTER_NAME="<YOUR_CLUSTER_NAME>"
REGION="us-central1"   <-- Replace with your actual REGION
GCP_USER_EMAIL="ABC@gmail.com"  # <-- Replace with your actual email
GCP_SERVICE_ACCOUNT="gke-lb-controller@$PROJECT_ID.iam.gserviceaccount.com"

# Install Kubeclt
sudo apt-get update
sudo apt-get install kubectl

echo "Installing GKE Auth Plugin..."
sudo apt-get update
sudo apt-get install google-cloud-sdk-gke-gcloud-auth-plugin -y

echo "Authenticating with GCP..."
gcloud auth login

echo "Fetching GKE cluster credentials..."
gcloud container clusters get-credentials "$CLUSTER_NAME" \
  --region "$REGION" \
  --project "$PROJECT_ID"

echo "Creating Kubernetes service account for LB Controller..."
kubectl create sa gke-lb-controller -n kube-system || true
kubectl annotate serviceaccount gke-lb-controller -n kube-system \
  "iam.gke.io/gcp-service-account=$GCP_SERVICE_ACCOUNT" --overwrite

echo "Installing Helm..."
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | \
  sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" \
  | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm -y

echo "Assigning cluster-admin role to $GCP_USER_EMAIL..."
kubectl create clusterrolebinding cluster-admin-binding \
  --clusterrole=cluster-admin \
  --user="$GCP_USER_EMAIL" || true

echo "Deploying ArgoCD..."
kubectl create ns argocd || true
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.4.7/manifests/install.yaml
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'

echo "Installing jq..."
sudo apt install jq -y

echo "Waiting for ArgoCD external IP..."
while true; do
  ARGOCD_SERVER=$(kubectl get svc argocd-server -n argocd -o json | jq -r '.status.loadBalancer.ingress[0].ip')
  if [[ "$ARGOCD_SERVER" != "null" && -n "$ARGOCD_SERVER" ]]; then
    break
  fi
  echo "Waiting for LoadBalancer IP..."
  sleep 10
done
echo "ArgoCD available at: $ARGOCD_SERVER"

echo "Fetching ArgoCD admin password..."
ARGO_PWD=$(kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d)
echo "ArgoCD password: $ARGO_PWD"

echo "Installing ingress-nginx via Helm..."
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
kubectl create namespace ingress-nginx || true

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --set controller.service.type=LoadBalancer \
  --set controller.publishService.enabled=true

echo "Setup complete."
```



🗂️ Repository Structure
```
.
├── Application-Code/
│   ├── frontend/
│   │   └── src/...
│   └── backend/
│       └── src/...
│
├── Jenkins-code/
│   ├── setupbackendapp/
│   ├── setupfrontendapp/
│   └── setupinfra/
│
├── Kubernetes-Manifests-file/
│   ├── Backend/
│   ├── Database/
│   ├── Frontend/
│   ├── ingress.yaml
│   └── managedCert.yaml
```
🚀 Deployment Steps
1. Build & Push Docker Images using Jenkins
If using your own images, build and push them to your registry (e.g., Artifact Registry, Docker Hub):


2. Apply Kubernetes Manifests from ArgoCD
Connect to the ArgoCD instance setup in the K8 namespace


3. Check Deployment

kubectl get deployments
kubectl get svc frontend backend
Wait for the frontend service to receive an external IP (LoadBalancer).

4. Test the App
Access the frontend IP address in your browser:

The UI should interact with the backend, displaying data fetched from the database.

🤝 Contributing
Contributions welcome! Please open an issue or pull request for:
