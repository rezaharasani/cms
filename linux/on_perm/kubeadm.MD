
## 1) High-level plan
1. Prepare OS on all nodes (update, disable swap, enable kernel settings).
2. Install container runtime (containerd) and configure it for Kubernetes.
3. Install `kubeadm`, `kubelet`, `kubectl`.
4. Initialize the control plane with `kubeadm init` on the master.
5. Install a CNI (I show Calico).
6. Join worker nodes with the `kubeadm join` command produced by the master (or generate one).
7. Deploy your program (build an `arm64` container image, push to a registry or load it onto nodes, create Deployment + Service).


## 2) Prep: networking & host basics (run on every node)
Assume these hostnames / IPs:

* `control-node` — 172.16.112.193
* `worker1` — 172.16.112.192
* `worker2` — 172.16.112.194
* `worker3` — 172.16.112.195

If your NAT network gives different IPs, use those; ensure each node can reach others on their IPs (ping). Add entries to /etc/hosts on all nodes:

```
> sudo tee -a /etc/hosts > /dev/null <<EOF
172.16.112.193 control-node
172.16.112.192 worker1
172.16.112.194 worker2
172.16.112.195 worker3
EOF
```


## 3) Common setup for every node (run as root or with sudo)
```
# update and essentials
> sudo apt update && sudo apt -y upgrade
> sudo apt -y install apt-transport-https ca-certificates curl gnupg lsb-release

# disable swap (Kubernetes requires no swap)
> sudo swapoff -a
# make swapoff permanent
> sudo sed -i '/ swap / s/^/#/' /etc/fstab

# required sysctl settings for bridged traffic and ip forwarding
> sudo tee /etc/sysctl.d/k8s.conf > /dev/null <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

> sudo sysctl --system
```


## 4) Install containerd (recommended) and configure cgroup driver
```
# install containerd
> sudo apt update
> sudo apt -y install containerd

# create default config and set systemd cgroup driver
> sudo mkdir -p /etc/containerd
> containerd config default | sudo tee /etc/containerd/config.toml

# Edit config.toml to use systemd cgroup driver:
> sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# restart containerd
> sudo systemctl restart containerd
> sudo systemctl enable containerd
```
You can check with:
```
> sudo crictl info   # if crictl installed
# or
> sudo systemctl status containerd
```


## 5) Install kubeadm, kubelet, kubectl (same on all nodes)
### add Kubernetes apt repo
```
> curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmour -o /usr/share/keyrings/k8s-archive-keyring.gpg

> echo "deb [signed-by=/usr/share/keyrings/k8s-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

> sudo apt update
> sudo apt -y install kubelet kubeadm kubectl
> sudo apt-mark hold kubelet kubeadm kubectl
> sudo apt autoremove
```
Confirm versions:
```
> kubeadm version
> kubelet --version
> kubectl version --client
```

## 6) Initialize control plane (only on `control-node`)
Pick a Pod network CIDR matching the CNI you’ll install. I’ll use Calico with `192.168.0.0/16` as a common example. If you prefer Flannel, change the CIDR to `10.244.0.0/16` and install Flannel manifest instead.

On `control-node`:
```
# pull control-plane images (optional but speeds things)
> sudo kubeadm config images pull

# initialize
> sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --apiserver-advertise-address=172.16.112.193
```

**Notes**:
* `--apiserver-advertise-address` = control node IP (change if different).
* The `kubeadm init` output includes the `kubeadm join` command you’ll run on workers. Save it.

Set up `kubectl` for a non-root user (on control-node):
```
> mkdir -p $HOME/.kube
> sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
> sudo chown $(id -u):$(id -g) $HOME/.kube/config
```


## 7) Install Calico CNI (on control-node using kubectl)
Apply Calico manifest (run on control-node where kubectl is configured). The manifest URL below is typical; if you prefer a specific version check Calico docs — but the kubectl apply below is the standard approach:

```
> kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml
kubectl get pods -n kube-system -w
```

(If you cannot reach GitHub due to NAT restrictions, download the YAML locally and `kubectl apply -f ./calico.yaml`.)

Wait for CNI pods to be `Running`:
```
> kubectl get pods -n kube-system -w
# Ctrl+C to stop watching
```


## 8) Join worker nodes
From the `kubeadm init` output you received earlier you'll have a `kubeadm join ... --token ... --discovery-token-ca-cert-hash sha256:...` command. Run that command on each worker:

```
> sudo kubeadm join 172.16.112.193:6443 \
  --token go3yg4.30tygwi1n8yol4mg \
  --discovery-token-ca-cert-hash sha256:6b79ace522e52ee0bdb1576333c82c567c2ef7e14fa32b79c7e71759e441a2ad

[preflight] Running pre-flight checks
...
...
Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

If you lost the join command, on control-node generate one:
```
> sudo kubeadm token create --print-join-command
```

After joining, verify on control-node. You should see control-node and worker1/worker2/worker3 in `Ready` state (may take a moment):
```
> kubectl get nodes
kube-master-01: Sat Sep 27 22:10:55 2025

NAME             STATUS   ROLES           AGE     VERSION
kube-master-01   Ready    control-plane   11h     v1.34.1
kube-worker-01   Ready    <none>          4m26s   v1.34.1
kube-worker-02   Ready    <none>          4m12s   v1.34.1
```

## 9) Common troubleshooting / requirements (important)
* **Firewall / ports**: ensure nodes can open these (control-plane<->workers and kubelet):
  * TCP 6443 (API server) reachable for joins
  * TCP/UDP 10250 (kubelet), 10251, 10252 (control plane components)
  * CNI-required ports (Calico uses BGP or IP-in-IP overlays; if using Calico IP-in-IP or VXLAN ensure overlay traffic is allowed)
* **Time sync**: ensure ntp/chrony so clocks are correct.
* **Swap** must be disabled.
* If on VMs with NAT: ensure the VMs can directly reach each other’s internal NAT IPs (VMware NAT networks generally do), and ensure no host firewall blocks traffic between VMs.


## 10) Allow scheduling on master? (optional)
By default the control-plane is tainted and pods won’t be scheduled on it. If you want to allow normal pods on the control node (not recommended for production), remove the taint:

```
kubectl taint nodes control-node node-role.kubernetes.io/control-plane:NoSchedule-   # or
kubectl taint nodes control-node node-role.kubernetes.io/master:NoSchedule-
```


## 11) Deploy your program
Two common ways to get an image onto cluster:
1. Build an `arm64` image and push to a public/private registry (Docker Hub, GitHub Container Registry, ECR, etc).
2. Build image locally and `docker save` + `ctr -n k8s.io images import` or `crictl` load onto each node (less convenient).

### Example: build & push arm64 image (locally or in an arm64 CI)
If you build on a non-arm64 machine, use docker buildx to build multi-arch:

```
# on your dev machine
docker buildx create --use
docker buildx build --platform linux/arm64 -t myregistry/myapp:1.0 --push .
```
Then on the cluster use a Deployment + NodePort Service example.

### Deployment YAML (example)
Save as `deployment.yaml`, edit image name:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myregistry/myapp:1.0   # <- change to your arm64 image
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: myapp-nodeport
spec:
  type: NodePort
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080   # access via nodeIP:30080
```

Apply:
```
kubectl apply -f deployment.yaml
kubectl get deployments,svc,pods
```
Access via `http://<worker-or-control-IP>:30080`.

If you want an HTTP Ingress and a domain, deploy an ingress controller (e.g., `nginx-ingress`) and configure DNS or use NodePort + port-forwarding.


## 12) Useful kubectl commands
```
kubectl get nodes
kubectl get pods -A
kubectl describe node worker1
kubectl logs deploy/myapp
kubectl port-forward svc/myapp-nodeport 8080:80   # local access to service
```


## 13) Backups & upgrades (short)
* Keep `admin.conf` safe for cluster admin tasks.
* To upgrade Kubernetes later: upgrade `kubeadm` → `kubeadm upgrade plan` → `kubeadm upgrade apply` → upgrade `kubelet` and `kubectl` on nodes, then restart kubelet. Test in a lab first.


## 14) Quick checklist (before starting)
 * [ ] All nodes have Debian 12 arm64 and internet access.
 * [ ] Swap disabled.
 * [ ] Hostnames & /etc/hosts entries set.
 * [ ] Time sync (chrony/ntp).
 * [ ] Ports between nodes open (6443, 10250, etc.) or NAT * [ ] rules permit VM<->VM traffic.
 * [ ] Your app image built for linux/arm64 and accessible from nodes (registry or pre-loaded).


## 15) Example common pitfalls & fixes
* **Worker stuck in NotReady**: check `kubectl describe node` and `journalctl -u kubelet -f` on worker. Often missing CNI or wrong `pod-network-cidr`.
* **Pods CrashLoopBackOff**: `kubectl logs podname`; check image architecture mismatch (x86 image on arm64 node will fail).
* **CNI pods pending**: check if kube-proxy or calico pods failing; also ensure `net.bridge.bridge-nf-call-iptables = 1`.