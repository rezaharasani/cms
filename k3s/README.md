# How to install K3s Kubenetes Cluster

## Install K3s (single-node)

```shell
curl -sfL https://get.k3s.io | sh -
# verify
sudo kubectl get nodes
```
K3s installs a single control-plane + worker by default.

Access kubeconfig:
```shell
sudo cat /etc/rancher/k3s/k3s.yaml
# copy to your local ~/.kube/config
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
```

Check:
```shell
kubectl get nodes
```


## Install K3s (multi-node)

### Master node:
```shell
curl -sfL https://get.k3s.io | sh -
sudo cat /var/lib/rancher/k3s/server/node-token
```

### Worker nodes:
```shell
curl -sfL https://get.k3s.io | K3S_URL=https://<MASTER_IP>:6443 K3S_TOKEN=<TOKEN> sh -
```

### Verify cluster
```shell
kubectl get nodes -o wide
kubectl get pods -A
```
