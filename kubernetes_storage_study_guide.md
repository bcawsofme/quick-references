# Kubernetes Storage Study Guide

Use this guide to understand Kubernetes storage options, how Pods request storage, how PersistentVolumes bind, and how to practice storage troubleshooting.

## Mental Model

| Term | Meaning | Study note |
| --- | --- | --- |
| Volume | Storage mounted into a Pod. | Can be temporary, config-backed, node-backed, or persistent. |
| PersistentVolume | Cluster-level storage object. | Often abbreviated PV. It represents actual storage. |
| PersistentVolumeClaim | A request for storage by a workload. | Often abbreviated PVC. Pods usually reference PVCs, not PVs. |
| StorageClass | Defines dynamic provisioning behavior. | Controls provisioner, parameters, reclaim policy, and binding mode. |
| CSI | Container Storage Interface. | Plugin system for storage providers. |
| Access mode | How storage may be mounted. | `ReadWriteOnce`, `ReadOnlyMany`, `ReadWriteMany`, or `ReadWriteOncePod`. |
| Reclaim policy | What happens to a PV after the PVC is deleted. | Usually `Delete` or `Retain`. |
| Volume mode | Whether a volume is exposed as a filesystem or block device. | `Filesystem` is the common default. |
| Dynamic provisioning | Automatic PV creation when a PVC requests a StorageClass. | Common in cloud and managed clusters. |
| Static provisioning | Admin manually creates PVs before claims use them. | Common in exam practice and fixed storage setups. |

## Storage Options Overview

| Option | Lifetime | Best for | Notes |
| --- | --- | --- | --- |
| `emptyDir` | Pod lifetime. | Scratch space, shared files between containers. | Deleted when Pod leaves the Node. |
| `configMap` volume | ConfigMap lifetime. | Mounting config files. | Non-sensitive data. |
| `secret` volume | Secret lifetime. | Mounting credentials or keys. | Sensitive data, still handle carefully. |
| `hostPath` | Node filesystem lifetime. | Node-level labs, agents, local testing. | Ties Pod to a Node; risky in production. |
| PVC-backed volume | Storage backend lifetime. | Persistent app data. | Normal app persistence pattern. |
| CSI volume | Provider dependent. | Cloud, enterprise, or external storage. | Uses a CSI driver. |
| Projected volume | Pod lifetime. | Combining Secret, ConfigMap, Downward API, ServiceAccount token. | Useful for app config bundles. |

## Volume Types You Should Recognize

| Volume type | What it does | CKA/CKAD relevance |
| --- | --- | --- |
| `emptyDir` | Creates an empty directory for a Pod. | Common multi-container practice. |
| `configMap` | Mounts ConfigMap keys as files. | CKAD config task. |
| `secret` | Mounts Secret keys as files. | CKAD security/config task. |
| `hostPath` | Mounts a path from the Node. | CKA node/storage practice. |
| `persistentVolumeClaim` | Mounts storage requested by a PVC. | Core CKA/CKAD storage topic. |
| `projected` | Combines multiple sources into one volume. | Useful to understand, less common in basic tasks. |
| `downwardAPI` | Exposes Pod metadata as files or env vars. | CKAD app config topic. |

## Access Modes

| Mode | Meaning | Good mental shortcut |
| --- | --- | --- |
| `ReadWriteOnce` | Read-write by one Node. | RWO: one Node can mount it read-write. |
| `ReadOnlyMany` | Read-only by many Nodes. | ROX: many readers, no writers. |
| `ReadWriteMany` | Read-write by many Nodes. | RWX: many readers and writers. |
| `ReadWriteOncePod` | Read-write by one Pod. | RWOP: stricter than RWO. |

Study notes:

```text
Access mode support depends on the storage backend.
A PVC can stay Pending if no PV or StorageClass can satisfy the requested mode.
RWO does not mean only one Pod. It means one Node for many common backends.
```

## PV and PVC Binding Flow

```text
Pod -> PVC -> PV -> real storage
```

| Step | What happens |
| --- | --- |
| 1 | A PVC requests size, access mode, and optionally a StorageClass. |
| 2 | Kubernetes finds a matching PV or dynamically provisions one. |
| 3 | The PVC binds to the PV. |
| 4 | A Pod mounts the PVC as a volume. |
| 5 | The kubelet mounts the storage on the Node for the container. |

Check the flow:

```bash
kubectl get pvc
kubectl describe pvc <claim>
kubectl get pv
kubectl describe pv <volume>
kubectl describe pod <pod>
```

## PersistentVolume

Static PV example:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /mnt/data
```

Study notes:

```text
PV is cluster-scoped.
Capacity, accessModes, storageClassName, and selectors affect binding.
hostPath is useful for labs but not a normal production storage backend.
```

## PersistentVolumeClaim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: manual
  resources:
    requests:
      storage: 1Gi
```

Use in a Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "echo hello > /data/hello.txt && sleep 3600"]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: data
```

Practice:

```bash
kubectl apply -f pv.yaml
kubectl apply -f pvc.yaml
kubectl apply -f pod.yaml
kubectl get pv,pvc,pod
kubectl exec app -- cat /data/hello.txt
```

## StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: example.com/csi
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
parameters:
  type: ssd
```

| Field | Meaning |
| --- | --- |
| `provisioner` | Storage plugin that creates volumes. |
| `reclaimPolicy` | Default reclaim behavior for dynamically created PVs. |
| `volumeBindingMode` | When binding/provisioning happens. |
| `allowVolumeExpansion` | Whether PVC resize is allowed. |
| `parameters` | Provider-specific options. |

## Binding Modes

| Mode | Meaning | Use when |
| --- | --- | --- |
| `Immediate` | Bind or provision as soon as PVC is created. | Storage is not topology-sensitive. |
| `WaitForFirstConsumer` | Wait until a Pod uses the PVC before binding or provisioning. | Storage depends on Node, zone, or topology. |

Study notes:

```text
WaitForFirstConsumer helps avoid provisioning storage in the wrong zone.
Pending PVCs with WaitForFirstConsumer may be normal until a Pod uses them.
```

## Reclaim Policies

| Policy | What happens when PVC is deleted | Use when |
| --- | --- | --- |
| `Delete` | PV and backend storage are deleted. | Dynamic cloud storage and disposable data. |
| `Retain` | PV and data remain. | Important data or manual recovery. |
| `Recycle` | Deprecated. | Know it is old; avoid using it. |

Practice:

```bash
kubectl get pv
kubectl delete pvc data
kubectl get pv
kubectl describe pv <volume>
```

## Volume Mounts

Mount full volume:

```yaml
volumeMounts:
  - name: data
    mountPath: /data
```

Mount one key or subdirectory:

```yaml
volumeMounts:
  - name: config
    mountPath: /etc/app/config.yaml
    subPath: config.yaml
```

Read-only mount:

```yaml
volumeMounts:
  - name: config
    mountPath: /etc/app
    readOnly: true
```

Study notes:

```text
subPath mounts do not always receive live updates from ConfigMaps and Secrets.
readOnly is useful for config and secret mounts.
```

## emptyDir

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared
spec:
  containers:
    - name: writer
      image: busybox
      command: ["sh", "-c", "while true; do date >> /cache/log.txt; sleep 5; done"]
      volumeMounts:
        - name: cache
          mountPath: /cache
    - name: reader
      image: busybox
      command: ["sh", "-c", "tail -f /cache/log.txt"]
      volumeMounts:
        - name: cache
          mountPath: /cache
  volumes:
    - name: cache
      emptyDir: {}
```

Practice:

```bash
kubectl apply -f emptydir.yaml
kubectl logs shared -c reader
kubectl delete pod shared
```

## ConfigMap and Secret Volumes

ConfigMap volume:

```yaml
volumes:
  - name: config
    configMap:
      name: app-config
```

Secret volume:

```yaml
volumes:
  - name: secret
    secret:
      secretName: app-secret
```

Practice:

```bash
kubectl create configmap app-config --from-literal=config.yaml='mode: practice'
kubectl create secret generic app-secret --from-literal=password=secret
kubectl describe configmap app-config
kubectl describe secret app-secret
```

## hostPath

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-demo
spec:
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "ls -la /host && sleep 3600"]
      volumeMounts:
        - name: host
          mountPath: /host
  volumes:
    - name: host
      hostPath:
        path: /tmp
        type: Directory
```

Study notes:

```text
hostPath ties the Pod to node-local data.
It can expose host files to containers.
It is useful for CKA practice but risky in production.
```

## StatefulSet Storage

StatefulSets often use `volumeClaimTemplates` so each Pod gets its own PVC.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: web
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: nginx
          volumeMounts:
            - name: data
              mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 1Gi
```

Practice:

```bash
kubectl apply -f statefulset-storage.yaml
kubectl get statefulset,pod,pvc
kubectl delete statefulset web
kubectl get pvc
```

Study notes:

```text
Deleting a StatefulSet does not automatically delete its PVCs.
Stable Pod identity plus stable storage is the point.
```

## Resizing PVCs

```bash
kubectl patch pvc data -p '{"spec":{"resources":{"requests":{"storage":"2Gi"}}}}'
kubectl get pvc data
kubectl describe pvc data
```

Requirements:

```text
StorageClass must allow expansion.
Storage provider must support expansion.
Filesystem resize may happen when the volume is mounted by a Pod.
```

## Troubleshooting Storage

```bash
kubectl get storageclass
kubectl describe storageclass <class>
kubectl get pv
kubectl describe pv <volume>
kubectl get pvc -A
kubectl describe pvc <claim>
kubectl describe pod <pod>
kubectl get events --sort-by=.metadata.creationTimestamp
```

| Symptom | Likely cause | Check |
| --- | --- | --- |
| PVC stuck Pending. | No matching PV, missing StorageClass, unsupported access mode, WaitForFirstConsumer. | `kubectl describe pvc`. |
| Pod stuck Pending. | PVC not bound or storage topology issue. | `kubectl describe pod`. |
| Mount failed. | Node cannot mount backend storage. | Pod events and node logs. |
| Permission denied. | Filesystem ownership or security context. | `runAsUser`, `fsGroup`, app UID. |
| Data disappeared. | Used `emptyDir` or deleted dynamic PVC with Delete policy. | Volume type and reclaim policy. |
| Config file not updated. | Used `subPath` or app does not reload config. | VolumeMounts and app behavior. |

## Practice Scenarios

### Scenario 1: Static PV and PVC Binding

Task:

```text
Create a hostPath PV, bind a PVC to it, mount it into a Pod, and write a file.
```

Commands:

```bash
kubectl apply -f pv.yaml
kubectl apply -f pvc.yaml
kubectl get pv,pvc
kubectl apply -f pod.yaml
kubectl exec app -- sh -c 'echo practice > /data/file.txt'
kubectl exec app -- cat /data/file.txt
```

What to learn:

```text
PV and PVC bind through size, access mode, and storageClassName.
Pods mount PVCs, not PVs directly.
```

### Scenario 2: Debug a Pending PVC

Task:

```text
Create a PVC that cannot bind. Find the reason and fix it.
```

Ideas:

```text
Request too much storage.
Use the wrong storageClassName.
Request an unsupported access mode.
Create a PVC before the matching PV exists.
```

Commands:

```bash
kubectl get pvc
kubectl describe pvc <claim>
kubectl get pv
kubectl describe pv <volume>
```

### Scenario 3: Compare emptyDir and PVC Persistence

Task:

```text
Write data to emptyDir and PVC-backed Pods. Delete the Pods and compare what survives.
```

Commands:

```bash
kubectl delete pod <emptydir-pod>
kubectl delete pod <pvc-pod>
kubectl apply -f pod.yaml
kubectl exec app -- ls -la /data
```

What to learn:

```text
emptyDir is tied to Pod lifetime.
PVC-backed storage survives Pod replacement.
```

### Scenario 4: StatefulSet PVCs

Task:

```text
Deploy a StatefulSet with volumeClaimTemplates, scale it, and inspect the PVCs.
```

Commands:

```bash
kubectl apply -f statefulset-storage.yaml
kubectl get pods,pvc
kubectl scale statefulset web --replicas=1
kubectl get pods,pvc
kubectl scale statefulset web --replicas=3
kubectl get pods,pvc
```

What to learn:

```text
Each StatefulSet replica gets its own PVC.
Scaling down does not delete the PVCs.
```

### Scenario 5: Fix Permission Denied

Task:

```text
Mount a volume into a Pod running as a non-root user. Fix write permission using securityContext.
```

Pattern:

```yaml
securityContext:
  runAsUser: 1000
  runAsGroup: 1000
  fsGroup: 1000
```

Commands:

```bash
kubectl describe pod <pod>
kubectl logs <pod>
kubectl exec <pod> -- id
kubectl exec <pod> -- ls -ld /data
```

## Exam and Interview Tips

| Question | Good answer |
| --- | --- |
| What object does a Pod reference for persistent storage? | PVC. |
| What object represents actual cluster storage? | PV. |
| What object enables dynamic provisioning? | StorageClass. |
| What does `Retain` do? | Keeps PV and data after PVC deletion. |
| What does `Delete` do? | Deletes dynamically provisioned storage after PVC deletion. |
| Why is PVC Pending? | No matching PV, bad StorageClass, unsupported access mode, topology wait, or provisioner issue. |
| What storage is deleted with the Pod? | `emptyDir`. |
| What is `volumeClaimTemplates` for? | Per-replica PVCs in StatefulSets. |
| What helps fix volume write permissions? | Pod/container securityContext, especially `fsGroup`. |
