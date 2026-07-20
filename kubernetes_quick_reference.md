# Kubernetes Quick Reference Guide

## kubectl Context and Config
```bash
kubectl version
kubectl cluster-info
kubectl config get-contexts
kubectl config current-context
kubectl config use-context <context>
kubectl config set-context --current --namespace=<namespace>
kubectl config view
kubectl api-resources
kubectl api-versions
```

## Namespaces
```bash
kubectl get namespaces
kubectl create namespace <namespace>
kubectl describe namespace <namespace>
kubectl delete namespace <namespace>
kubectl get all -n <namespace>
kubectl get all -A
```

## Get and Describe Resources
```bash
kubectl get pods
kubectl get pods -o wide
kubectl get pods -A
kubectl get deployments
kubectl get services
kubectl get ingress
kubectl get configmaps
kubectl get secrets
kubectl get nodes
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl describe pod <pod>
kubectl describe deployment <deployment>
kubectl describe service <service>
kubectl describe node <node>
```

## Apply, Edit, and Delete
```bash
kubectl apply -f file.yaml
kubectl apply -f ./manifests/
kubectl apply -k overlays/dev
kubectl diff -f file.yaml
kubectl edit deployment <deployment>
kubectl replace -f file.yaml
kubectl delete -f file.yaml
kubectl delete pod <pod>
kubectl delete deployment <deployment>
kubectl delete service <service>
```

## Pods and Logs
```bash
kubectl logs <pod>
kubectl logs <pod> -c <container>
kubectl logs -f <pod>
kubectl logs --previous <pod>
kubectl exec -it <pod> -- /bin/bash
kubectl exec -it <pod> -- /bin/sh
kubectl cp <namespace>/<pod>:/path/file ./file
kubectl top pod
kubectl top pod -A
```

## Deployments and Rollouts
```bash
kubectl rollout status deployment/<deployment>
kubectl rollout restart deployment/<deployment>
kubectl rollout history deployment/<deployment>
kubectl rollout undo deployment/<deployment>
kubectl scale deployment <deployment> --replicas=3
kubectl set image deployment/<deployment> <container>=<image>:<tag>
kubectl autoscale deployment <deployment> --min=2 --max=10 --cpu-percent=80
```

## Services, Ingress, and Networking
```bash
kubectl get svc
kubectl describe svc <service>
kubectl expose deployment <deployment> --port=80 --target-port=8080
kubectl port-forward pod/<pod> 8080:80
kubectl port-forward svc/<service> 8080:80
kubectl get endpoints
kubectl get ingress
kubectl describe ingress <ingress>
kubectl get networkpolicy
```

## ConfigMaps and Secrets
```bash
kubectl create configmap <name> --from-literal=KEY=value
kubectl create configmap <name> --from-file=config.yaml
kubectl get configmap <name> -o yaml
kubectl create secret generic <name> --from-literal=KEY=value
kubectl create secret generic <name> --from-file=secret.txt
kubectl get secret <name> -o yaml
kubectl describe secret <name>
```

## Nodes and Cluster Health
```bash
kubectl get nodes
kubectl get nodes -o wide
kubectl describe node <node>
kubectl cordon <node>
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
kubectl uncordon <node>
kubectl top node
kubectl get componentstatuses
```

## Storage
```bash
kubectl get storageclass
kubectl get pv
kubectl get pvc
kubectl describe pvc <claim>
kubectl describe pv <volume>
kubectl delete pvc <claim>
```

## RBAC and Service Accounts
```bash
kubectl get serviceaccounts
kubectl get roles
kubectl get rolebindings
kubectl get clusterroles
kubectl get clusterrolebindings
kubectl auth can-i get pods
kubectl auth can-i create deployments --as=<user>
kubectl create serviceaccount <name>
kubectl create role <role> --verb=get,list,watch --resource=pods
kubectl create rolebinding <binding> --role=<role> --serviceaccount=<namespace>:<name>
```

## Jobs and CronJobs
```bash
kubectl get jobs
kubectl get cronjobs
kubectl create job <job> --image=<image>
kubectl create job <job> --from=cronjob/<cronjob>
kubectl logs job/<job>
kubectl delete job <job>
```

## Kustomize
```bash
kubectl kustomize overlays/dev
kubectl apply -k overlays/dev
kubectl diff -k overlays/dev
kubectl delete -k overlays/dev
```

## Helm
```bash
helm repo add <name> <url>
helm repo update
helm search repo <chart>
helm install <release> <chart>
helm upgrade <release> <chart>
helm upgrade --install <release> <chart>
helm list
helm list -A
helm status <release>
helm history <release>
helm rollback <release> <revision>
helm uninstall <release>
helm template <release> <chart>
```

## Minikube CKA Practice Setup

### Install on Linux
```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube_latest_amd64.deb
sudo dpkg -i minikube_latest_amd64.deb
minikube version
```

### Start a two-node practice cluster
```bash
minikube start -p cka --nodes=2 --driver=docker --container-runtime=containerd --cpus=2 --memory=4096
kubectl config use-context cka
kubectl get nodes -o wide
minikube status -p cka
```

### Start with a specific Kubernetes version
```bash
minikube start -p cka --nodes=2 --driver=docker --container-runtime=containerd --kubernetes-version=<version>
minikube start -p cka --kubernetes-version=v1.34.0
```

### Use kubectl from Minikube
```bash
minikube kubectl -- get pods -A
alias kubectl="minikube kubectl --"
kubectl get pods -A
```

### Useful CKA addons
```bash
minikube addons list -p cka
minikube addons enable metrics-server -p cka
minikube addons enable ingress -p cka
minikube addons enable storage-provisioner-rancher -p cka
kubectl top nodes
kubectl get pods -A
kubectl get storageclass
```

### Node access and images
```bash
minikube ssh -p cka
minikube ssh -p cka -n cka-m02
minikube image ls -p cka
minikube image load <image>:<tag> -p cka
```

### Practice service access
```bash
kubectl create deployment web --image=nginx
kubectl expose deployment web --type=NodePort --port=80
kubectl get svc web
minikube service web -p cka
kubectl port-forward svc/web 8080:80
```

### Stop, delete, and rebuild
```bash
minikube stop -p cka
minikube start -p cka
minikube delete -p cka
minikube delete --all
```

## kubeadm
```bash
kubeadm version
sudo kubeadm init
sudo kubeadm init --pod-network-cidr=<cidr>
sudo kubeadm token create --print-join-command
sudo kubeadm join <api-server>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
sudo kubeadm reset
sudo kubeadm upgrade plan
sudo kubeadm upgrade apply <version>
```

## kubeadm One Control Plane and One Worker

### Run on both nodes
```bash
sudo swapoff -a
sudo modprobe overlay
sudo modprobe br_netfilter
sudo sysctl -w net.bridge.bridge-nf-call-iptables=1
sudo sysctl -w net.bridge.bridge-nf-call-ip6tables=1
sudo sysctl -w net.ipv4.ip_forward=1
sudo systemctl enable --now containerd
sudo systemctl enable --now kubelet
```

### Run on the control plane node
```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
kubeadm token create --print-join-command
```

### Run on the worker node
```bash
sudo kubeadm join <control-plane-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

### Verify from the control plane node
```bash
kubectl get nodes
kubectl get pods -A
kubectl describe node <worker-node>
```

### Reset a node
```bash
sudo kubeadm reset
sudo rm -rf /etc/cni/net.d
sudo rm -rf $HOME/.kube
```

## Troubleshooting
```bash
kubectl get events -A --sort-by=.metadata.creationTimestamp
kubectl describe pod <pod>
kubectl logs <pod> --previous
kubectl get pod <pod> -o yaml
kubectl get deployment <deployment> -o yaml
kubectl get endpoints <service>
kubectl run debug --rm -it --image=busybox -- /bin/sh
kubectl debug node/<node> -it --image=busybox
kubectl explain pod
kubectl explain deployment.spec.template.spec.containers
```

## Useful Output Formats
```bash
kubectl get pods -o wide
kubectl get pods -o yaml
kubectl get pods -o json
kubectl get pods -o name
kubectl get pods --show-labels
kubectl get pods -l app=<label>
kubectl get pods --field-selector=status.phase=Running
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
```
