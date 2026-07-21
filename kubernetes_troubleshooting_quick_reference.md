# Kubernetes Troubleshooting Quick Reference Guide

## First Look
```bash
kubectl get pods -A
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.metadata.creationTimestamp
kubectl get nodes -o wide
kubectl top nodes
kubectl top pods -A
```

## Pod Troubleshooting
```bash
kubectl describe pod <pod>
kubectl logs <pod>
kubectl logs <pod> -c <container>
kubectl logs <pod> --previous
kubectl get pod <pod> -o yaml
kubectl exec -it <pod> -- /bin/sh
kubectl delete pod <pod>
```

## Common Pod States
```text
Pending             check node capacity, taints, affinity, PVCs
ImagePullBackOff    check image name, tag, registry auth
CrashLoopBackOff    check logs, command, args, probes, config
CreateContainerError check secrets, configmaps, volume mounts
Running not ready   check readinessProbe and endpoints
Terminating stuck   check finalizers and node health
```

## Deployment and Rollout
```bash
kubectl describe deployment <deployment>
kubectl rollout status deployment/<deployment>
kubectl rollout history deployment/<deployment>
kubectl rollout undo deployment/<deployment>
kubectl get rs
kubectl describe rs <replicaset>
kubectl scale deployment <deployment> --replicas=<n>
```

## Services and Endpoints
```bash
kubectl get svc
kubectl describe svc <service>
kubectl get endpoints <service>
kubectl get pods -l <selector>
kubectl port-forward svc/<service> 8080:80
kubectl run curl --image=curlimages/curl --rm -it --restart=Never -- curl http://<service>:<port>
```

## DNS
```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns
kubectl run dns-test --image=busybox --rm -it --restart=Never -- nslookup kubernetes.default
kubectl run dns-test --image=busybox --rm -it --restart=Never -- nslookup <service>.<namespace>.svc.cluster.local
```

## Nodes
```bash
kubectl describe node <node>
kubectl get node <node> -o yaml
kubectl cordon <node>
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
kubectl uncordon <node>
kubectl get events --field-selector involvedObject.kind=Node
```

## Cluster Components
```bash
kubectl get pods -n kube-system
kubectl logs -n kube-system <pod>
kubectl get componentstatuses
kubectl get --raw /readyz
kubectl get --raw /livez
kubectl get --raw /metrics
```

## kubeadm Host Checks
```bash
sudo systemctl status kubelet
sudo journalctl -u kubelet -n 100 --no-pager
sudo crictl ps
sudo crictl images
sudo crictl logs <container-id>
sudo crictl inspect <container-id>
```

## Storage
```bash
kubectl get pv
kubectl get pvc -A
kubectl describe pvc <claim>
kubectl describe pv <volume>
kubectl get storageclass
kubectl describe pod <pod>
```

## NetworkPolicy
```bash
kubectl get networkpolicy -A
kubectl describe networkpolicy <policy>
kubectl run client --image=busybox --rm -it --restart=Never -- wget -qO- http://<service>:<port>
```
