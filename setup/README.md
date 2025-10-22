# Nirmata Remediator Agent Installation Guide

## 🧩 Required Components

- **Kubernetes Cluster:** Running Kubernetes **v1.20+**  
- **Helm:** Version **3.x** installed and configured  
- **kubectl:** Configured to access your cluster  

---

## 🔐 Authentication Requirements

- **Nirmata API Token:** Your personal **NCH token**  
  - If you don’t have an account, [sign up for a 15-day free trial](https://www.nirmata.io/free-trial) to obtain your API token.

---

## ⚡ Quick Installation

### 1. Create Namespace and Secrets

```bash
# Create namespace
kubectl create namespace nirmata

# Create Nirmata API token secret
kubectl create secret generic nirmata-api-token   --from-literal=api-token=YOUR_NIRMATA_API_TOKEN   --namespace nirmata
```

---

### 2. Install the Remediator Agent

Add and update the Helm repository:

```bash
helm repo add nirmata https://nirmata.github.io/kyverno-charts
helm repo update nirmata
```

Install the Helm chart:

```bash
helm install remediator nirmata/remediator-agent --devel   --namespace nirmata   --create-namespace   --set nirmata.apiTokenSecret="nirmata-api-token"
```

---

### 3. Configure Git Credentials

Follow the steps mentioned in the official Nirmata documentation to configure Git credentials:  
👉 [Using Nirmata App or Personal Access Token](https://docs.nirmata.io/docs/agents/service-agents/remediator/tools/#using-personal-access-token)

---

## 🧱 Local Cluster Mode

You can also specify the repository-to-namespace mappings using a `ConfigMap`.

### 1. Create the ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: repo-namespace-mapping
  namespace: nirmata
data:
  mapping: |
    [
      {
        "repo": "https://github.com/nirmata/demo-remediator",
        "branch": "main",
        "path": "apps/nginx",
        "targetNamespace": "default"
      }
    ]
```

Apply the ConfigMap:

```bash
kubectl apply -f repo-namespace-mapping.yaml
```

---

### 2. Apply the Remediator CR

```yaml
apiVersion: serviceagents.nirmata.io/v1alpha1
kind: Remediator
metadata:
  name: remediator-local-cluster
  namespace: nirmata
spec:
  environment:
    type: localCluster
  
  target:
    localCluster:
      repoNamespaceMappingRef:
        name: repo-namespace-mapping
        namespace: nirmata
        key: mapping

  remediation:
    triggers:
      - schedule:
          crontab: "0 */6 * * *"

    llmConfigRef:
      name: remediator-agent-llm
      namespace: nirmata

    gitCredentials:
      name: toolconfig-sample
      namespace: nirmata

    actions:
      - type: CreatePR
        toolRef:
          name: toolconfig-sample
          namespace: nirmata
```

Apply the manifest:

```bash
kubectl apply -f remediator-local-cluster.yaml
```

---

## ✅ Verification

Once deployed, verify the agent status:

```bash
kubectl get pods -n nirmata
```

You should see the **remediator-agent** running successfully.

---




# 🚀 Steps to Deploy and Configure ArgoCD

## Step 1: Deploy ArgoCD

### 1.1 Create ArgoCD Namespace
```bash
kubectl create namespace argocd
```

### 1.2 Install ArgoCD
```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 1.3 Wait for ArgoCD to be Ready
```bash
kubectl wait --for=condition=available --timeout=300s deployment/argocd-server -n argocd
```

## Step 2: Create the Nginx Demo Application

### 2.1 Deploy the Application
```bash
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-demo
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/nirmata/demo-remediation-argo
    targetRevision: main
    path: demo-remediator-main/apps/nginx
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
EOF
```

## Step 3: Access ArgoCD UI

### 3.1 Set up Port Forwarding
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

### 3.2 Get Admin Password
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

### 3.3 Access ArgoCD
- **URL**: https://localhost:8080
- **Username**: `admin`
- **Password**: (output from step 3.2)

## Step 4: Verify Application Sync

1. Open your browser and navigate to https://localhost:8080
2. Login with the credentials from step 3.2
3. You should see the `nginx-demo` application in the ArgoCD UI
4. The application will automatically sync the nginx demo from the specified repository

## Managing Auto-Sync

You can disable AUTO-SYNC by clicking on the application in ArgoCD → Navigate to details and hover at the bottom under sync policy.
