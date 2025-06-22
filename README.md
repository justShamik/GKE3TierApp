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
# Install GKE Auth Plugin
sudo apt-get install google-cloud-sdk-gke-gcloud-auth-plugin -y

# Authenticate with GCP
gcloud auth login

# Fetch GKE Cluster Credentials
gcloud container clusters get-credentials democluster \
  --region us-central1 \
  --project white-welder-463307-r7

# Create a Kubernetes service account for GKE Load Balancer controller
kubectl create sa gke-lb-controller -n kube-system
kubectl annotate serviceaccount gke-lb-controller -n kube-system \
  iam.gke.io/gcp-service-account=gke-lb-controller@white-welder-463307-r7.iam.gserviceaccount.com

# Install Helm
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" \
  | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm -y

# Assign cluster-admin role (use your email)
kubectl create clusterrolebinding cluster-admin-binding \
  --clusterrole=cluster-admin \
  --user=<youremail>

# Configure ArgoCD
kubectl create ns argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.4.7/manifests/install.yaml
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'

# Install jq (for parsing ArgoCD outputs)
sudo apt install jq -y

# Get ArgoCD server details
export ARGOCD_SERVER=$(kubectl get svc argocd-server -n argocd -o json | jq -r '.status.loadBalancer.ingress[0].hostname')
export ARGO_PWD=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)
echo $ARGO_PWD

# Install ingress-nginx controller via Helm
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

kubectl create namespace ingress-nginx

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --set controller.service.type=LoadBalancer \
  --set controller.publishService.enabled=true
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
