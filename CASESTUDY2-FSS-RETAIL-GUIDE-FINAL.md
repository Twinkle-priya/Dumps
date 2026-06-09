# 📦 CaseStudy2 — FSS Retail App Assessment Guide
**Repo:** https://github.com/nanineelapu/fss-Retail-App_kubernetes  
**Branch:** master (NOT main!)  
**Cluster:** fssCluster (NEW cluster — no Helm!)

---

> 🟢 **DONE / SAME AS CS1** = Already know how, quick verify  
> 🔴 **MUST DO** = Fresh work needed  
> 📸 **SCREENSHOT HERE** = Take this screenshot now  
> ⚠️ **IMPORTANT** = Don't miss this

---

## 📁 REPO STRUCTURE (What's inside)

```
fss-Retail-App_kubernetes/
├── .github/workflows/     ← CI pipeline already exists!
├── k8s-manifests/         ← YOUR DEPLOYMENT FILES (no Helm!)
├── Public/                ← Frontend static files
├── node_modules/          ← Node.js dependencies
├── Dockerfile             ← Docker build file
├── server.js              ← Node.js app
├── package.json           ← App dependencies
├── docker-compose.yaml    ← Local dev only
└── .env                   ← Environment variables
```

⚠️ **NO HELM CHART** — This uses direct Kubernetes YAML manifests from `k8s-manifests/` folder

---

## 🏗️ ARCHITECTURE FLOW

```
Developer pushes code
        ↓
GitHub Repo (Msocial123/fss-Retail-App_kubernetes) [branch: master]
        ↓
GitHub Actions CI (.github/workflows/)
        ↓
Docker Image → ECR (fss-retail-app)
        ↓
k8s-manifests/ updated with new image tag
        ↓
ArgoCD detects Git change → auto-syncs
        ↓
Amazon EKS (fssCluster → fss-retail namespace)
        ↓
Datadog Monitoring
        ↓
AI Troubleshooting Agent (Groq)
```

---

## ✅ STEP 1 — Create NEW EKS Cluster (fssCluster)
🔴 **MUST DO — Takes 15-20 minutes**

```bash
# Create fresh cluster for CaseStudy2
eksctl create cluster \
  --name fssCluster \
  --region ap-southeast-2 \
  --nodegroup-name fss-nodes \
  --node-type t3.small \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 2 \
  --managed

# Wait 15-20 minutes until complete...

# Connect to new cluster
aws eks update-kubeconfig --name fssCluster --region ap-southeast-2

# Verify
kubectl get nodes
```

📸 **SCREENSHOT:** `kubectl get nodes` showing 2 nodes as `Ready`

---

## ✅ STEP 2 — Create Namespaces
🔴 **MUST DO**

```bash
# Create all required namespaces
kubectl create namespace fss-retail
kubectl create namespace argocd
kubectl create namespace argo-rollouts

# Verify
kubectl get ns
```

📸 **SCREENSHOT:** `kubectl get ns` showing all namespaces

---

## ✅ STEP 3 — Install ArgoCD
🟢 **SAME AS CS1**

```bash
# Install ArgoCD
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait 2 minutes
sleep 120

# Check pods
kubectl get pods -n argocd

# Expose as LoadBalancer
kubectl patch svc argocd-server -n argocd \
  -p '{"spec": {"type": "LoadBalancer"}}'

# Wait for EXTERNAL-IP (1-2 minutes)
kubectl get svc -n argocd

# Get ArgoCD password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
# SAVE THIS PASSWORD!

# Login
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
sleep 3
argocd login localhost:8080 --username admin --insecure
```

📸 **SCREENSHOT:**
- `kubectl get pods -n argocd` all Running
- `kubectl get svc -n argocd` showing EXTERNAL-IP

---

## ✅ STEP 4 — Fork & Clone FSS Retail Repo
🔴 **MUST DO**

```bash
# FIRST: Fork the repo on GitHub browser
# Go to: https://github.com/Msocial123/fss-Retail-App_kubernetes
# Click FORK → fork to nanineelapu account

# Then clone YOUR fork
cd ~
git clone https://github.com/nanineelapu/fss-Retail-App_kubernetes.git
cd fss-Retail-App_kubernetes

# IMPORTANT: branch is master not main!
git checkout master

# Check repo structure
ls -la
ls -la k8s-manifests/
```

📸 **SCREENSHOT:** `ls -la` and `ls -la k8s-manifests/`

---

## ✅ STEP 5 — Check k8s Manifests & Deploy App
🔴 **MUST DO — No Helm, direct kubectl apply!**

```bash
cd ~/fss-Retail-App_kubernetes

# Check what manifests exist
ls -la k8s-manifests/
cat k8s-manifests/*.yml 2>/dev/null || cat k8s-manifests/*.yaml 2>/dev/null

# Check what Docker image is used
grep -r "image:" k8s-manifests/

# Deploy directly with kubectl (NO helm!)
kubectl apply -f k8s-manifests/ -n fss-retail

# Wait for pods
sleep 60
kubectl get pods -n fss-retail
kubectl get svc -n fss-retail
kubectl get all -n fss-retail
```

📸 **SCREENSHOT:**
- `kubectl get pods -n fss-retail` showing Running pods
- `kubectl get svc -n fss-retail` showing services with EXTERNAL-IP

---

## ✅ STEP 6 — Setup ArgoCD GitOps (No Helm!)
🔴 **MUST DO — Different from CS1 (no Helm values!)**

```bash
# Create ArgoCD Application manifest
cat > ~/fss-argocd-app.yml << 'YAML'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: fss-retail-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/nanineelapu/fss-Retail-App_kubernetes.git
    targetRevision: master
    path: k8s-manifests
  destination:
    server: https://kubernetes.default.svc
    namespace: fss-retail
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
YAML

# Apply it
kubectl apply -f ~/fss-argocd-app.yml

# Wait for sync
sleep 30

# Verify
argocd app list
argocd app get fss-retail-app
```

⚠️ **Key differences from CS1:**
- `path: k8s-manifests` (not Retail-chart)
- `targetRevision: master` (not main!)
- No `helm:` section at all

📸 **SCREENSHOT:**
- `argocd app list` showing fss-retail-app Synced + Healthy
- ArgoCD UI browser showing fss-retail-app

---

## ✅ STEP 7 — GitHub Actions CI Pipeline
🔴 **MUST DO — Check existing workflow first**

```bash
# Check if workflow already exists in the repo
cat ~/fss-Retail-App_kubernetes/.github/workflows/*.yml 2>/dev/null

# If workflow exists, just add AWS secrets to GitHub
# Go to: https://github.com/nanineelapu/fss-Retail-App_kubernetes/settings/secrets/actions
# Add:
# - AWS_ACCESS_KEY_ID
# - AWS_SECRET_ACCESS_KEY
# - DOCKERHUB_USERNAME (if using DockerHub)
# - DOCKERHUB_TOKEN (if using DockerHub)
```

If NO workflow exists, create one:

```bash
mkdir -p ~/fss-Retail-App_kubernetes/.github/workflows

cat > ~/fss-Retail-App_kubernetes/.github/workflows/ci-cd.yml << 'EOF'
name: FSS Retail CI/CD

on:
  push:
    branches: [ master ]
  pull_request:
    branches: [ master ]

env:
  AWS_REGION: ap-southeast-2
  ECR_REPOSITORY: fss-retail-app
  IMAGE_TAG: ${{ github.sha }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout
      uses: actions/checkout@v3

    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'

    - name: Install dependencies
      run: npm install

    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ env.AWS_REGION }}

    - name: Login to Amazon ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v1

    - name: Build and Push Docker image
      run: |
        docker build -t ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ env.IMAGE_TAG }} .
        docker push ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ env.IMAGE_TAG }}

    - name: Update k8s manifest with new image
      run: |
        sed -i "s|image:.*|image: ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ env.IMAGE_TAG }}|g" k8s-manifests/*.yaml || true
        sed -i "s|image:.*|image: ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ env.IMAGE_TAG }}|g" k8s-manifests/*.yml || true

    - name: Commit updated manifest
      run: |
        git config --global user.email "ci@fss.com"
        git config --global user.name "CI Bot"
        git add k8s-manifests/ || true
        git commit -m "Update image: ${{ env.IMAGE_TAG }}" || true
        git push origin master || true
EOF

# Create ECR repository
aws ecr create-repository \
  --repository-name fss-retail-app \
  --region ap-southeast-2

# Commit and push workflow
cd ~/fss-Retail-App_kubernetes
git add .github/workflows/ci-cd.yml
git commit -m "Add CI/CD pipeline"
git push origin master

# Trigger workflow by making a small change
echo "# Updated $(date)" >> README.md
git add README.md
git commit -m "Trigger CI/CD pipeline test"
git push origin master
```

📸 **SCREENSHOT:**
- GitHub Actions tab showing workflow triggered and running
- Green checkmark showing workflow completed

---

## ✅ STEP 8 — Datadog Monitoring
🟢 **SAME AS CS1**

```bash
# Add Datadog Helm repo (if not already added)
helm repo add datadog https://helm.datadoghq.com
helm repo update

# Install Datadog on fssCluster
# REPLACE YOUR_API_KEY with your actual Datadog API key
helm install datadog datadog/datadog \
  --namespace datadog \
  --create-namespace \
  --set datadog.apiKey=YOUR_API_KEY \
  --set datadog.site=datadoghq.com \
  --set datadog.logs.enabled=true \
  --set datadog.logs.containerCollectAll=true \
  --set datadog.apm.enabled=true \
  --set datadog.processAgent.enabled=true \
  --set agents.rbac.create=true \
  --set clusterAgent.enabled=true

# Wait and verify
sleep 60
kubectl get pods -n datadog
```

📸 **SCREENSHOT:**
- `kubectl get pods -n datadog` showing agents Running
- Datadog UI showing fss-retail pods

---

## ✅ STEP 9 — AI Troubleshooting Agent
🟢 **SAME SCRIPT — test with fss-retail pods**

```bash
# Set Groq API key
export GROQ_API_KEY="your-groq-key-here"

# Get fss-retail pod name first
kubectl get pods -n fss-retail

# Run AI agent
python3 ~/ai-troubleshooting-agent.py

# When prompted, enter:
# fss-retail/<pod-name-from-above>
```

📸 **SCREENSHOT:** Terminal showing AI analyzing fss-retail pod + AI response

---

## ✅ STEP 10 — Final Verification
🔴 **DO BEFORE SUBMITTING**

```bash
# Full status check
echo "=== CLUSTER NODES ==="
kubectl get nodes

echo "=== ALL NAMESPACES ==="
kubectl get ns

echo "=== FSS RETAIL PODS ==="
kubectl get pods -n fss-retail

echo "=== FSS RETAIL SERVICES ==="
kubectl get svc -n fss-retail

echo "=== ARGOCD PODS ==="
kubectl get pods -n argocd

echo "=== ARGOCD APPS ==="
argocd app list

echo "=== DATADOG PODS ==="
kubectl get pods -n datadog

echo "=== HELM RELEASES ==="
helm list -A
```

---

## 📸 COMPLETE SCREENSHOT CHECKLIST

| # | What to Screenshot | Command / Where |
|---|-------------------|-----------------|
| 1 | 🟢 Cluster nodes Ready | `kubectl get nodes` |
| 2 | 🟢 All namespaces | `kubectl get ns` |
| 3 | 🟢 ArgoCD pods Running | `kubectl get pods -n argocd` |
| 4 | 🟢 ArgoCD service with EXTERNAL-IP | `kubectl get svc -n argocd` |
| 5 | 🟢 FSS Retail pods Running | `kubectl get pods -n fss-retail` |
| 6 | 🟢 FSS Retail services | `kubectl get svc -n fss-retail` |
| 7 | 🟢 ArgoCD app Synced+Healthy | `argocd app list` |
| 8 | 🟢 ArgoCD UI browser | Open ArgoCD EXTERNAL-IP in browser |
| 9 | 🟢 GitHub Actions ran | GitHub → Actions tab |
| 10 | 🟢 Datadog pods | `kubectl get pods -n datadog` |
| 11 | 🟢 Datadog UI | https://app.datadoghq.com |
| 12 | 🟢 AI Agent demo | Terminal showing pod analysis |
| 13 | 🟢 Repo k8s-manifests | `ls -la k8s-manifests/` |
| 14 | 🟢 App in browser | Open app EXTERNAL-IP in browser |

---

## ⚡ QUICK START TOMORROW (All Commands in Order)

```bash
# 1. Create cluster (15-20 min)
eksctl create cluster --name fssCluster --region ap-southeast-2 \
  --nodegroup-name fss-nodes --node-type t3.small --nodes 2 --managed

# 2. Connect
aws eks update-kubeconfig --name fssCluster --region ap-southeast-2
kubectl get nodes

# 3. Create namespaces
kubectl create namespace fss-retail
kubectl create namespace argocd

# 4. Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
sleep 120
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
sleep 3
argocd login localhost:8080 --username admin --insecure

# 5. Clone repo
cd ~
git clone https://github.com/nanineelapu/fss-Retail-App_kubernetes.git
cd fss-Retail-App_kubernetes
git checkout master

# 6. Deploy app (NO Helm! Direct kubectl)
kubectl apply -f k8s-manifests/ -n fss-retail
sleep 30
kubectl get pods -n fss-retail

# 7. Apply ArgoCD app
kubectl apply -f ~/fss-argocd-app.yml
sleep 20
argocd app list

# 8. Install Datadog
helm install datadog datadog/datadog \
  --namespace datadog --create-namespace \
  --set datadog.apiKey=YOUR_API_KEY \
  --set datadog.site=datadoghq.com \
  --set datadog.logs.enabled=true \
  --set datadog.logs.containerCollectAll=true

# 9. Run AI Agent
export GROQ_API_KEY="your-key"
python3 ~/ai-troubleshooting-agent.py
```

---

## ⚠️ KEY DIFFERENCES FROM CASESTUDY1

| Item | CaseStudy1 (Shine) | CaseStudy2 (FSS Retail) |
|------|-------------------|------------------------|
| Cluster | shineCluster | fssCluster (NEW) |
| Repo | nanineelapu/Shine-GitOps | nanineelapu/fss-Retail-App |
| Branch | main | **master** ⚠️ |
| Namespace | retail | fss-retail |
| Deploy method | **Helm chart** | **Direct kubectl apply** ⚠️ |
| ArgoCD path | Retail-chart | **k8s-manifests** ⚠️ |
| App type | Node.js userprofile | Node.js retail app |
| CI/CD | Already created | Use existing or create new |

---

## 💰 COST REMINDER

```bash
# DELETE fssCluster AFTER assessment!
eksctl delete cluster --name fssCluster --region ap-southeast-2
```

Both clusters running = ~$0.68/hr. Delete immediately after assessment!

---

*Good luck tomorrow! 🚀*
