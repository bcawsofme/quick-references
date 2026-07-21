# CKA and CKAD Exam Speed Sheet

## Shell Setup
```bash
alias k=kubectl
alias kgp='kubectl get pods'
alias kga='kubectl get all'
alias kaf='kubectl apply -f'
alias kdf='kubectl delete -f'
export do='--dry-run=client -o yaml'
export now='--force --grace-period=0'
```

## Vim Setup
```vim
:set number
:set expandtab
:set tabstop=2
:set shiftwidth=2
:set paste
```

## Context and Namespace
```bash
kubectl config get-contexts
kubectl config use-context <context>
kubectl config current-context
kubectl config set-context --current --namespace=<namespace>
kubectl create namespace <namespace>
```

## Generate YAML Fast
```bash
kubectl run nginx --image=nginx $do > pod.yaml
kubectl create deployment web --image=nginx $do > deploy.yaml
kubectl expose deployment web --port=80 --target-port=8080 $do > svc.yaml
kubectl create configmap app-config --from-literal=MODE=prod $do > cm.yaml
kubectl create secret generic app-secret --from-literal=TOKEN=secret $do > secret.yaml
```

## Explain API Fields
```bash
kubectl explain pod
kubectl explain pod.spec
kubectl explain pod.spec.containers
kubectl explain deployment.spec.strategy
kubectl explain networkpolicy.spec
kubectl explain persistentvolumeclaim.spec
```

## Fast Checks
```bash
kubectl get pods -A
kubectl get pods -o wide
kubectl describe pod <pod>
kubectl logs <pod>
kubectl logs <pod> --previous
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl top pods
kubectl top nodes
```

## Rollouts
```bash
kubectl set image deployment/<deployment> <container>=<image>:<tag>
kubectl rollout status deployment/<deployment>
kubectl rollout history deployment/<deployment>
kubectl rollout undo deployment/<deployment>
kubectl scale deployment <deployment> --replicas=<n>
```

## Services
```bash
kubectl expose deployment web --port=80 --target-port=8080
kubectl get svc
kubectl get endpoints
kubectl port-forward svc/web 8080:80
kubectl run curl --image=curlimages/curl --rm -it --restart=Never -- curl http://web
```

## RBAC Checks
```bash
kubectl auth can-i get pods
kubectl auth can-i create deployments --as=<user>
kubectl create serviceaccount <name>
kubectl create role pod-reader --verb=get,list,watch --resource=pods
kubectl create rolebinding read-pods --role=pod-reader --serviceaccount=<namespace>:<name>
```

## Cleanup
```bash
kubectl delete pod <pod> $now
kubectl delete deployment <deployment>
kubectl delete svc <service>
kubectl delete job <job>
kubectl delete namespace <namespace>
```

## Time Management
```text
Read the task twice.
Set the correct context first.
Use imperative commands to generate YAML.
Apply, verify, then move on.
Skip and return if blocked after a few minutes.
```
