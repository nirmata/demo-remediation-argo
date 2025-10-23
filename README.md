# Demo Remediation Agent 

See: https://docs.google.com/document/d/1eurCPTuj7Nik04Smr40BQaWyVNm3iGPNOKz4hQhFA-0/edit?tab=t.0


<details>
  <summary>Installation Guide</summary>

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

```sh
kubectl apply -f config/cm-repo-namespace-mapping.yaml
```

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

---

### 2. Apply the Remediator CR

```sh
kubectl apply -f config/cm-repo-namespace-mapping.yaml
```

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
kubectl apply -f apps/nginx/
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

Note, make sure the application is already running and in-sync:


# Install & Configure Kyverno

```sh
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update
helm install kyverno kyverno/kyverno -n kyverno --create-namespace --set 
```

Patch kyverno to skip the `nirmata` and `argocd` namespaces:

```sh
kubectl patch configmap kyverno -n kyverno --type merge -p '{"data":{"webhooks": "{\\"namespaceSelector\\":{\\"matchExpressions\\":[{\\"key\\":\\"kubernetes.io/metadata.name\\",\\"operator\\":\\"NotIn\\",\\"values\\":[\\"kube-system\\",\\"kyverno\\",\\"nirmata\\",\\"argocd\\"]}],\\"matchLabels\\":null}}"}}'
```

Check the webhooks:

```sh
kubectl -n kyverno get validatingwebhookconfigurations kyverno-resource-validating-webhook-cfg -o yaml
```

Restart Kyverno:

```sh
kubectl -n kyverno rollout restart deployments
```

## Install Policies

This installs the pod security policy set. You can choose your own policies:

```sh
kustomize build https://github.com/nirmata/kyverno-policies/pod-security/enforce | kubectl apply -f -
```

## Set ClusterPolicy allowExistingViolations to false

This ensures that existing violations remain unchanged and are not updated or deleted when any resource is modified.

```sh
kubectl get clusterpolicy -o json | jq '(.items[] | .spec.rules[] | select(.validate != null) | .validate).allowExistingViolations = false' | kubectl apply -f -
```

Verify using:

```sh
kubectl get clusterpolicy -o json | jq -r '.items[] | . as $policy | .spec.rules[]? | select(.validate != null) | [$policy.metadata.name, .name, .validate.allowExistingViolations] | @tsv'
```

## Generate Policy Reports

These reports are utilized by the remediation agent to create a pull request (PR).

Verify that policy reports are generated:

```sh
kubectl get polr -n nginx
```

## Switch Policies to Enforce Mode

Update the policies mode to enforce, so the application will be blocked when there is an error:

```sh
kubectl get clusterpolicy -o name | xargs -I {} kubectl patch {} --type=merge -p '{"spec":{"validationFailureAction":"Enforce"}}'
```

## Redeploy the Application

Redeploy the application (delete the pod/resource using kubectl and then hit sync in ArgoCD) and you should see ArgoCD blocking the deployment due to policy enforcement.

</details>
</p></p>

# Demo Script

## Connect to the ArgoCD UI:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

### 3.3 Access ArgoCD
- **URL**: https://localhost:8080
- **Username**: `admin`
- **Password**: (output from step 3.2)

The Application should be in failed state showing policy violations.

## Trigger the Remediation Agent

Trigger the remediation agent by deleting its pod:

```sh
kubectl -n nirmata rollout restart deployment deploy/remediator-agent
```

This will restart the agent and initiate the remediation process.

## View the logs

```sh
kubectl -n nirmata rollout restart deployment deploy/remediator-agent
```

## Check the GitHub repo for a PR



