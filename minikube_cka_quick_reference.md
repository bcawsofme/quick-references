# Minikube CKA Practice Quick Reference Guide

## Install on Linux
```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube_latest_amd64.deb
sudo dpkg -i minikube_latest_amd64.deb
minikube version
```

## Start a Two-Node Practice Cluster
```bash
minikube start -p cka --nodes=2 --driver=docker --container-runtime=containerd --cpus=2 --memory=4096
kubectl config use-context cka
kubectl get nodes -o wide
minikube status -p cka
```

## Start with a Specific Kubernetes Version
```bash
minikube start -p cka --nodes=2 --driver=docker --container-runtime=containerd --kubernetes-version=<version>
minikube start -p cka --kubernetes-version=v1.34.0
```

## Use kubectl from Minikube
```bash
minikube kubectl -- get pods -A
alias kubectl="minikube kubectl --"
kubectl get pods -A
```

## Useful CKA Addons
```bash
minikube addons list -p cka
minikube addons enable metrics-server -p cka
minikube addons enable ingress -p cka
minikube addons enable storage-provisioner-rancher -p cka
kubectl top nodes
kubectl get pods -A
kubectl get storageclass
```

## Node Access and Images
```bash
minikube ssh -p cka
minikube ssh -p cka -n cka-m02
minikube image ls -p cka
minikube image load <image>:<tag> -p cka
```

## Practice Service Access
```bash
kubectl create deployment web --image=nginx
kubectl expose deployment web --type=NodePort --port=80
kubectl get svc web
minikube service web -p cka
kubectl port-forward svc/web 8080:80
```

## CKA Practice Tasks
```bash
kubectl create namespace practice
kubectl create deployment app --image=nginx -n practice
kubectl scale deployment app --replicas=3 -n practice
kubectl expose deployment app --port=80 --type=ClusterIP -n practice
kubectl run debug --rm -it --image=busybox -- /bin/sh
kubectl create configmap app-config --from-literal=MODE=practice -n practice
kubectl create secret generic app-secret --from-literal=TOKEN=example -n practice
```

## Stop, Delete, and Rebuild
```bash
minikube stop -p cka
minikube start -p cka
minikube delete -p cka
minikube delete --all
```
