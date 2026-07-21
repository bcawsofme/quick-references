# kubectl Imperative Quick Reference Guide

## Dry Run Pattern
```bash
kubectl run <name> --image=<image> --dry-run=client -o yaml
kubectl create deployment <name> --image=<image> --dry-run=client -o yaml
kubectl create service clusterip <name> --tcp=80:8080 --dry-run=client -o yaml
kubectl create configmap <name> --from-literal=KEY=value --dry-run=client -o yaml
kubectl create secret generic <name> --from-literal=KEY=value --dry-run=client -o yaml
```

## Pods
```bash
kubectl run nginx --image=nginx
kubectl run busybox --image=busybox --restart=Never -- sleep 3600
kubectl run curl --image=curlimages/curl --restart=Never -it --rm -- sh
kubectl run debug --image=busybox --restart=Never -it --rm -- /bin/sh
kubectl delete pod <pod>
```

## Deployments
```bash
kubectl create deployment web --image=nginx
kubectl create deployment web --image=nginx --replicas=3
kubectl scale deployment web --replicas=5
kubectl set image deployment/web nginx=nginx:1.27
kubectl rollout status deployment/web
kubectl rollout history deployment/web
kubectl rollout undo deployment/web
kubectl delete deployment web
```

## Services
```bash
kubectl expose pod nginx --port=80 --target-port=80 --name=nginx
kubectl expose deployment web --port=80 --target-port=8080
kubectl expose deployment web --type=NodePort --port=80
kubectl create service clusterip web --tcp=80:8080
kubectl create service nodeport web --tcp=80:8080
kubectl create service loadbalancer web --tcp=80:8080
```

## ConfigMaps and Secrets
```bash
kubectl create configmap app-config --from-literal=MODE=prod
kubectl create configmap app-config --from-file=config.yaml
kubectl create secret generic app-secret --from-literal=TOKEN=secret
kubectl create secret generic app-secret --from-file=password.txt
kubectl create secret tls tls-secret --cert=tls.crt --key=tls.key
```

## Jobs and CronJobs
```bash
kubectl create job oneoff --image=busybox -- date
kubectl create job manual --from=cronjob/<cronjob>
kubectl create cronjob backup --image=busybox --schedule="*/5 * * * *" -- date
kubectl logs job/oneoff
kubectl delete job oneoff
```

## Labels and Annotations
```bash
kubectl label pod <pod> app=web
kubectl label pod <pod> app-
kubectl label deployment <deployment> tier=frontend --overwrite
kubectl annotate pod <pod> owner=platform
kubectl annotate pod <pod> owner-
kubectl get pods -l app=web
kubectl get pods --show-labels
```

## Namespaces
```bash
kubectl create namespace <namespace>
kubectl config set-context --current --namespace=<namespace>
kubectl get all -n <namespace>
kubectl delete namespace <namespace>
```

## Fast Edit Flow
```bash
kubectl create deployment web --image=nginx --dry-run=client -o yaml > deploy.yaml
kubectl run app --image=nginx --dry-run=client -o yaml > pod.yaml
kubectl expose deployment web --port=80 --dry-run=client -o yaml > service.yaml
kubectl apply -f deploy.yaml
```
