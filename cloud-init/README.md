## Launch customized instances with Multipass and cloud-init

You can set up instances with a customized environment or configuration using the launch command along with a custom cloud-init YAML file and an optional post-launch health check to ensure everything is working correctly.


#### 🐳 docker:
Launch with:
```
multipass launch 24.04 \
  --name docker \
  --cpus 2 \
  --memory 4G \
  --disk 20G \
  --cloud-init https://github.com/rezaharasani/cms/blob/main/cloud-init/docker-cloud-init.yml 
```

Health check:
```
multipass exec docker -- bash -c "docker run hello-world"
```

You can also optionally add aliases:
```
multipass prefer docker
multipass alias docker:docker docker
multipass alias docker:docker-compose docker-compose
multipass prefer default
multipass aliases
```


#### ☸️ minikube:
Launch with:
```
multipass launch \
  --name minikube \
  --cpus 2 \
  --memory 4G \
  --disk 20G \
  --timeout 1800 \
  --cloud-init https://github.com/rezaharasani/cms/blob/main/cloud-init/minikube-cloud-init.yml
```

Health check:
```
multipass exec minikube -- bash -c "set -e
  minikube status
  kubectl cluster-info"
```
