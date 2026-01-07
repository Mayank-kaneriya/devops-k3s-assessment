# Open WebUI on k3s with Helm & OIDC

This README documents the **end‑to‑end project**:

* Install Docker on EC2
* Install & configure **k3s (Kubernetes)**
* Install **Helm v3**
* Deploy **Open WebUI** using Helm
* Configure **OIDC authentication**
* Access the app
* Debug common issues
* Production readiness Q&A (interview‑ready)

---

## 1. Prerequisites

* Ubuntu EC2 instance
* SSH access with sudo
* Open ports (22, 80/443 as needed)

---

## 2. Install Docker (APT)

sudo apt update
sudo apt install -y ca-certificates curl gnupg

sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io

sudo usermod -aG docker $USER
newgrp docker

docker version
---



## 3. Install k3s (Kubernetes)

bash
curl -sfL https://get.k3s.io | sh -

sudo systemctl status k3s

kubectl get nodes
--------------------------------------------------------------------------

Expected output:

```text
NAME              STATUS   ROLES           VERSION
<node-name>       Ready    control-plane   v1.x+k3s
```

---

## 4. Install Helm v3

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

helm version
```

Verify Helm ↔ Kubernetes:

```bash
helm list
```

---

## 5. Add Open WebUI Helm Repo

```bash
helm repo add open-webui https://helm.openwebui.com
helm repo update
```

---

## 6. Deploy Open WebUI (Dry Run)

```bash
kubectl create namespace openwebui

helm install webui open-webui/open-webui \
  --namespace openwebui \
  --set service.type=ClusterIP \
  --dry-run
```

---

## 7. Deploy Open WebUI (Actual Install)

```bash
helm install webui open-webui/open-webui \
  --namespace openwebui \
  --set service.type=ClusterIP
```

Verify:

```bash
kubectl get pods -n openwebui
kubectl get svc  -n openwebui
```

---

## 8. Access the Application

### Option A: Port Forward (Recommended)

```bash
kubectl port-forward svc/open-webui 8080:80 -n openwebui
```

Open browser:

```
http://localhost:8080
```

### Option B: Inside Cluster (curl)

```bash
kubectl run curlpod --rm -it --image=curlimages/curl -- /bin/sh

curl http://open-webui.openwebui.svc.cluster.local:80
```

---

## 9. Configure OIDC Authentication

### 9.1 Create `values-oidc.yaml`

```yaml
oidc:
  enabled: true
  clientId: "test"
  issuer: "https://<OIDC-DOMAIN>/.well-known/openid-configuration"
  scopes:
    - openid
    - profile
    - email
```

> ⚠️ Client secret should **NOT** be in Git

### 9.2 Create OIDC Secret

```bash
kubectl create secret generic webui-oidc \
  -n openwebui \
  --from-literal=clientSecret=XXXXX
```

### 9.3 Upgrade Helm Release

```bash
helm upgrade webui open-webui/open-webui \
  -n openwebui \
  -f values-oidc.yaml
```

---

## 10. Custom CA for Self‑Signed OIDC Provider

### Create ConfigMap with CA

```bash
kubectl create configmap oidc-ca \
  -n openwebui \
  --from-file=ca.crt=custom-ca.pem
```

### Helm Values (No Code Changes)

```yaml
extraVolumes:
  - name: oidc-ca
    configMap:
      name: oidc-ca

extraVolumeMounts:
  - name: oidc-ca
    mountPath: /etc/ssl/custom-ca
    readOnly: true

extraEnv:
  - name: NODE_EXTRA_CA_CERTS
    value: /etc/ssl/custom-ca/ca.crt
```

---

## 11. Debugging Guide

### Pods not starting

```bash
kubectl get pods -n openwebui
kubectl describe pod <pod> -n openwebui
kubectl logs <pod> -n openwebui
```

### Helm issues

```bash
helm status webui -n openwebui
helm get values webui -n openwebui
helm history webui -n openwebui
```

### OIDC failures

```bash
kubectl logs deploy/open-webui -n openwebui
```

Common errors:

* `x509: unknown authority` → missing CA
* `401 / 403` → clientId / issuer mismatch

### Kubernetes unreachable

```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```

---

## 12. Failure Scenario (10× Traffic Spike)

### What breaks first

* Application pods (CPU / OOM)
* Node resources
* Kubernetes API responsiveness

### Recovery steps

* Restart node / k3s
* Scale down non‑critical pods
* Verify services

### Next‑day fixes

* Add second node
* Resource limits & probes
* Monitoring & alerts

---

## 13. Security & Secrets (Answers)

### Secret types

* OIDC client secrets
* API tokens
* Database passwords
* TLS private keys
* Service account tokens
* SSH keys

### Never in Git

* Secrets, tokens, passwords
* `.env` files
* TLS private keys

---

## 14. Backup & Recovery (Answers)

### What to back up

* Databases & Redis
* Persistent volumes
* Kubernetes manifests
* Helm values
* Secrets

### Recovery testing

* Restore in staging namespace
* Verify app & data
* Run periodic drills

---

## 15. Cost Control (Answers)

* Use k3s instead of managed K8s
* Small EC2 instances
* Avoid early HA & paid LBs
* Scale only what’s needed

### When to move away from k3s

* Need HA control plane
* High traffic & autoscaling
* Compliance requirements

---



✅ **End of README**
