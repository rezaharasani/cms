# Setup Rancher-K3s-Argocd

Here’s a step-by-step guide for installing Rancher on an Ubuntu Linux host, connecting it to a local K3s cluster, and 
then integrating it with Argo CD.


## 🧩 Overview
We’ll cover:
 1. System requirements
 2. Install K3s
 3. Install Rancher
 4. Access Rancher UI
 5. Connect Argo CD
 6. Optional — Secure with Ingress + HTTPS


## 1. System Requirements
Minimum recommended setup:
 * OS: Ubuntu 22.04 LTS or later
 * CPU: 2 vCPU or more
 * RAM: 4 GB or more
 * Storage: 20 GB
 * Network: Access to the internet for pulling container images

Update your system:
```shell
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget vim apt-transport-https ca-certificates
```

## 2. Install K3s – Local Kubernetes Cluster
Install a lightweight Kubernetes (K3s):
```shell
curl -sfL https://get.k3s.io | sh -
```

Verify installation:
```shell
sudo kubectl get nodes

```
Get kubeconfig:
```shell
sudo cat /etc/rancher/k3s/k3s.yaml
```

Copy it to your user’s kube config:
```shell
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
```

Check cluster:
```shell
kubectl get nodes -o wide
```

## 3. Install Rancher
### Step 1: Add Helm repo for Rancher
```shell
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm repo update
```

### Step 2: Create a namespace for Rancher
```shell
kubectl create namespace cattle-system
```

### Step 3: Install cert-manager (for Rancher’s HTTPS)
```shell
helm repo add jetstack https://charts.jetstack.io
helm repo update

kubectl create namespace cert-manager
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v1.15.0 \
  --set installCRDs=true
```

Verify:
```shell
kubectl get pods -n cert-manager
```

### Step 4: Install Rancher with Helm
You can use a local hostname (like rancher.local) or IP address.
Example:
```shell
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=rancher.local \
  --set replicas=1
```

Wait for Rancher to start:
```shell
kubectl -n cattle-system rollout status deploy/rancher
```

## 4. Access the Rancher UI
If you used `rancher.local`, add it to your `/etc/hosts` file:
```shell
sudo bash -c 'echo "127.0.0.1 rancher.local" >> /etc/hosts'
```

Forward port 443 to your local machine:
```shell
kubectl -n cattle-system port-forward svc/rancher 8443:443
```

Then visit in your browser:  
👉 https://rancher.local:8443

Get the initial admin password:
```shell
kubectl get secret --namespace cattle-system bootstrap-secret \
  -o go-template='{{.data.bootstrapPassword|base64decode}}{{"\n"}}'
```

Login with username `admin` and that password.


## 5. Install and Connect Argo CD
To see the full ArgoCD installation documentaion, Please 
read [ArgoCD Installation Guide](https://github.com/rezaharasani/cms/blob/main/argocd/README.md) section. Then, follow 
the below instructions to connecto to ArgoCD by Rancher.


You can manage both tools together in two common ways in order to connect Rancher to ArgoCD:

### Option A: Manage Rancher via Argo CD (GitOps)
Add Rancher Helm chart to Argo CD as an Application:
 * Repository: https://releases.rancher.com/server-charts/latest
 * Chart: rancher
 * Namespace: cattle-system
 * Values: include your Rancher config (hostname, replicas, etc.)

### Option B: Manage Argo CD via Rancher
 * Go to **Cluster → Apps → Deploy**, and add the Argo CD Helm chart from within Rancher’s UI.
 * Then Rancher can control lifecycle and visibility of Argo CD.


## 6. Optional: Secure Access with Ingress and HTTPS
If you have a DNS domain and Let’s Encrypt access:
```shell
helm upgrade --install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=rancher.yourdomain.com \
  --set ingress.tls.source=letsEncrypt \
  --set letsEncrypt.email=you@yourdomain.com \
  --set letsEncrypt.ingress.class=traefik
```


