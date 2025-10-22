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
👉 [Using Personal Access Token](https://docs.nirmata.io/docs/agents/service-agents/remediator/tools/#using-personal-access-token)

---

## 🧱 Local Cluster Mode

If you are **not using ArgoCD** for deployments, specify the repository-to-namespace configuration using a `ConfigMap`.

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
