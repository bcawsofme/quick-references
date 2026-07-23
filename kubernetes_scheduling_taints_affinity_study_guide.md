# Kubernetes Scheduling, Taints, Tolerations, and Affinity Study Guide

Use this guide to study how Kubernetes decides where Pods run, how to force or prefer placement, and how to debug Pending Pods. This is important for CKA, CKAD, platform engineering, and SRE work because scheduling problems often look like application failures until you inspect the Pod events.

## Mental Model

```text
Pod created
  -> scheduler watches unscheduled Pod
  -> filters Nodes that cannot run it
  -> scores remaining Nodes
  -> binds Pod to selected Node
  -> kubelet starts containers on that Node
```

Common scheduling filters:

```text
Node readiness
Taints without matching tolerations
Node selectors and node affinity
Pod affinity and anti-affinity
Resource requests
Volume topology
Node ports
Topology spread constraints
```

## Core Terms

| Term | Meaning | Study note |
| --- | --- | --- |
| kube-scheduler | Control-plane component that assigns unscheduled Pods to Nodes. | It sets `spec.nodeName` after choosing a Node. |
| Scheduling | Selecting a Node for a Pod. | Happens before kubelet starts containers. |
| Binding | Writing the scheduler's Node choice to the Pod. | A bound Pod has `spec.nodeName`. |
| Pending Pod | Pod that has not started or is waiting on scheduling/resources. | Always check `kubectl describe pod`. |
| Node label | Key-value metadata on a Node. | Used by `nodeSelector`, affinity, and topology. |
| Taint | Node rule that repels Pods. | Node says "do not schedule here unless tolerated." |
| Toleration | Pod rule that allows scheduling onto a matching tainted Node. | It allows placement; it does not force placement. |
| `nodeSelector` | Simple hard Node label requirement. | Fastest exam tool for simple placement. |
| Node affinity | More expressive Node label requirement/preference. | Supports required and preferred rules. |
| Pod affinity | Places Pods near Pods with matching labels. | Uses topology keys such as hostname or zone. |
| Pod anti-affinity | Keeps Pods away from Pods with matching labels. | Useful for spreading replicas. |
| Topology spread constraint | Spreads Pods across topology domains. | More direct spreading control than anti-affinity. |
| Resource request | CPU/memory needed for scheduling. | Scheduler uses requests, not actual usage. |
| Volume topology | Storage placement requirement. | PVCs can force Pods into certain zones or Nodes. |

## Scheduling Decision Table

| Need | Use | Hard or soft |
| --- | --- | --- |
| Put Pod only on Nodes with a label. | `nodeSelector` | Hard |
| Put Pod only on Nodes matching complex label rules. | Required node affinity | Hard |
| Prefer certain Nodes, but allow fallback. | Preferred node affinity | Soft |
| Keep general workloads off special Nodes. | Taints on Nodes | Hard unless tolerated |
| Allow a Pod onto tainted Nodes. | Tolerations on Pods | Permission, not placement |
| Put Pods near other Pods. | Pod affinity | Hard or soft |
| Keep replicas apart. | Pod anti-affinity or topology spread | Hard or soft |
| Spread replicas across zones/nodes evenly. | Topology spread constraints | Hard or soft |
| Reserve capacity for scheduling. | Resource requests | Hard |

## Fast Debugging Commands

```bash
kubectl get pod <pod> -o wide
kubectl describe pod <pod>
kubectl get events --sort-by=.lastTimestamp
kubectl get nodes -o wide
kubectl describe node <node>
kubectl get nodes --show-labels
kubectl get pod <pod> -o yaml
```

Useful filters:

```bash
kubectl get pods -A --field-selector=status.phase=Pending
kubectl get pods -A -o wide | grep Pending
kubectl describe pod <pod> | sed -n '/Events:/,$p'
```

Typical Pending event messages:

| Event text | Likely issue |
| --- | --- |
| `Insufficient cpu` | Pod requests more CPU than available. |
| `Insufficient memory` | Pod requests more memory than available. |
| `node(s) had untolerated taint` | Pod lacks a matching toleration. |
| `node(s) didn't match Pod's node affinity/selector` | Node labels do not match. |
| `pod has unbound immediate PersistentVolumeClaims` | PVC is not bound. |
| `volume node affinity conflict` | Storage is tied to a different topology. |
| `max node group size reached` | Cluster autoscaler cannot add capacity. |

## Node Labels

Show labels:

```bash
kubectl get nodes --show-labels
kubectl describe node <node>
```

Add or remove labels:

```bash
kubectl label node <node> disk=ssd
kubectl label node <node> workload=apps
kubectl label node <node> zone=west
kubectl label node <node> disk-
```

Good practice labels:

| Label | Example | Purpose |
| --- | --- | --- |
| `workload` | `apps`, `platform`, `batch` | Separate workload types. |
| `disk` | `ssd`, `hdd` | Place storage-sensitive Pods. |
| `topology.kubernetes.io/zone` | `us-east-1a` | Zone-aware scheduling. |
| `kubernetes.io/hostname` | Node name | Node-level topology domain. |
| `node-role.kubernetes.io/control-plane` | empty value | Identifies control-plane Nodes. |

## nodeSelector

Use `nodeSelector` when you need a simple hard requirement.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-ssd
spec:
  nodeSelector:
    disk: ssd
  containers:
  - name: nginx
    image: nginx:1.27
```

Practice:

```bash
kubectl label node <node> disk=ssd
kubectl apply -f pod.yaml
kubectl get pod nginx-ssd -o wide
```

Troubleshoot:

```bash
kubectl describe pod nginx-ssd
kubectl get nodes -l disk=ssd
```

Exam note: `nodeSelector` is quick and usually enough when the question says "schedule this Pod on nodes labeled X."

## Taints

Taints live on Nodes and repel Pods.

```bash
kubectl taint node <node> dedicated=platform:NoSchedule
kubectl taint node <node> dedicated=platform:NoExecute
kubectl taint node <node> spot=true:PreferNoSchedule
```

Remove taints:

```bash
kubectl taint node <node> dedicated=platform:NoSchedule-
kubectl taint node <node> spot=true:PreferNoSchedule-
```

Taint effects:

| Effect | Meaning | Typical use |
| --- | --- | --- |
| `NoSchedule` | New Pods will not schedule unless they tolerate the taint. | Dedicated Nodes. |
| `PreferNoSchedule` | Scheduler tries to avoid the Node, but can still use it. | Soft separation. |
| `NoExecute` | New Pods will not schedule and existing Pods can be evicted unless tolerated. | Evict workloads from unhealthy or dedicated Nodes. |

View taints:

```bash
kubectl describe node <node> | grep -i taints
kubectl get node <node> -o jsonpath='{.spec.taints}'
```

## Tolerations

Tolerations live on Pods and allow Pods to run on matching tainted Nodes.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: platform-tool
spec:
  tolerations:
  - key: dedicated
    operator: Equal
    value: platform
    effect: NoSchedule
  containers:
  - name: nginx
    image: nginx:1.27
```

Operator examples:

| Toleration | Meaning |
| --- | --- |
| `operator: Equal` with key/value/effect | Tolerates that exact taint. |
| `operator: Exists` with key/effect | Tolerates any value for that key and effect. |
| Empty key with `operator: Exists` | Tolerates all taints with the matching effect. Use carefully. |
| `tolerationSeconds` | For `NoExecute`, controls how long Pod can stay before eviction. |

`NoExecute` with grace period:

```yaml
tolerations:
- key: maintenance
  operator: Equal
  value: planned
  effect: NoExecute
  tolerationSeconds: 300
```

Important: a toleration does not force the Pod onto the tainted Node. To force placement, combine a toleration with `nodeSelector` or node affinity.

## Dedicated Node Pattern

Goal: only platform workloads should run on platform Nodes.

Label and taint the Node:

```bash
kubectl label node <node> workload=platform
kubectl taint node <node> dedicated=platform:NoSchedule
```

Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: platform-tool
spec:
  nodeSelector:
    workload: platform
  tolerations:
  - key: dedicated
    operator: Equal
    value: platform
    effect: NoSchedule
  containers:
  - name: nginx
    image: nginx:1.27
```

Why both are needed:

| Field | Purpose |
| --- | --- |
| Taint | Keeps other Pods away from the Node. |
| Toleration | Allows this Pod onto the tainted Node. |
| `nodeSelector` | Attracts this Pod to the dedicated Node. |

## Node Affinity

Node affinity is more expressive than `nodeSelector`.

Hard node affinity:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disk
            operator: In
            values:
            - ssd
            - nvme
  containers:
  - name: nginx
    image: nginx:1.27
```

Soft node affinity:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-preferred-zone
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80
        preference:
          matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values:
            - us-east-1a
  containers:
  - name: nginx
    image: nginx:1.27
```

Operators:

| Operator | Meaning |
| --- | --- |
| `In` | Label value must be in the values list. |
| `NotIn` | Label value must not be in the values list. |
| `Exists` | Label key must exist. |
| `DoesNotExist` | Label key must not exist. |
| `Gt` | Label value must be greater than value. |
| `Lt` | Label value must be less than value. |

Study note: `IgnoredDuringExecution` means the Pod is not evicted if labels change after scheduling.

## Pod Affinity

Pod affinity schedules a Pod near other Pods that match labels.

Example: schedule a sidecar-like helper on the same Node as Pods labeled `app=web`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: helper-near-web
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: web
        topologyKey: kubernetes.io/hostname
  containers:
  - name: nginx
    image: nginx:1.27
```

Topology key meanings:

| Topology key | Meaning |
| --- | --- |
| `kubernetes.io/hostname` | Same Node. |
| `topology.kubernetes.io/zone` | Same zone. |
| Custom topology label | Depends on Node labels. |

Debug:

```bash
kubectl get pods --show-labels -o wide
kubectl describe pod helper-near-web
kubectl get nodes --show-labels
```

## Pod Anti-Affinity

Pod anti-affinity keeps Pods away from matching Pods.

Example: spread replicas across Nodes:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: web
              topologyKey: kubernetes.io/hostname
      containers:
      - name: nginx
        image: nginx:1.27
```

Hard anti-affinity:

```yaml
podAntiAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
  - labelSelector:
      matchLabels:
        app: web
    topologyKey: kubernetes.io/hostname
```

Warning: hard anti-affinity can make Pods Pending if there are not enough topology domains.

## Topology Spread Constraints

Topology spread constraints are often clearer than anti-affinity for spreading replicas.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 4
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: web
      containers:
      - name: nginx
        image: nginx:1.27
```

Fields:

| Field | Meaning |
| --- | --- |
| `maxSkew` | Maximum allowed imbalance between topology domains. |
| `topologyKey` | Node label used as the spread domain. |
| `whenUnsatisfiable: DoNotSchedule` | Hard rule. |
| `whenUnsatisfiable: ScheduleAnyway` | Soft preference. |
| `labelSelector` | Which Pods count when calculating spread. |

## Control-Plane Node Scheduling

Control-plane Nodes are often tainted.

Check:

```bash
kubectl describe node <control-plane-node> | grep -i taints
```

Common taint:

```text
node-role.kubernetes.io/control-plane:NoSchedule
```

To schedule a Pod there for practice, add a matching toleration:

```yaml
tolerations:
- key: node-role.kubernetes.io/control-plane
  operator: Exists
  effect: NoSchedule
```

Do not remove control-plane taints casually on real clusters. In labs, it can be acceptable if the exercise specifically asks for it.

## Resources and Scheduling

Scheduler uses requests to decide if a Node has room.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cpu-heavy
spec:
  containers:
  - name: stress
    image: nginx:1.27
    resources:
      requests:
        cpu: "4"
        memory: "2Gi"
      limits:
        cpu: "4"
        memory: "2Gi"
```

Debug insufficient resources:

```bash
kubectl describe pod cpu-heavy
kubectl describe node <node>
kubectl top nodes
kubectl top pods -A
```

Note: `kubectl top` shows live usage. Scheduling decisions use requested resources and allocatable capacity.

## Storage and Scheduling

PVCs can affect scheduling, especially with topology-aware storage.

Check:

```bash
kubectl describe pod <pod>
kubectl get pvc
kubectl describe pvc <pvc>
kubectl get pv
kubectl describe pv <pv>
```

Common storage scheduling issues:

| Symptom | Likely issue |
| --- | --- |
| `pod has unbound immediate PersistentVolumeClaims` | PVC is not bound yet. |
| `volume node affinity conflict` | PV is tied to a different Node or zone. |
| Pod waits until scheduled before PVC binds. | StorageClass uses `WaitForFirstConsumer`. |

## Practice Scenarios

### Scenario 1: Force a Pod onto a Labeled Node

```bash
kubectl label node <node> lab=scheduling
```

Create a Pod with:

```yaml
nodeSelector:
  lab: scheduling
```

Verify:

```bash
kubectl get pod <pod> -o wide
kubectl get nodes -l lab=scheduling
```

Goal: Pod lands on the labeled Node.

### Scenario 2: Create and Fix an Untolerated Taint

Taint a Node:

```bash
kubectl taint node <node> dedicated=platform:NoSchedule
```

Create a Pod with `nodeSelector` targeting that Node label but no toleration. It should stay Pending.

Fix by adding:

```yaml
tolerations:
- key: dedicated
  operator: Equal
  value: platform
  effect: NoSchedule
```

Goal: understand that selection plus toleration are separate controls.

### Scenario 3: Prefer SSD Nodes but Allow Fallback

Label one Node:

```bash
kubectl label node <node> disk=ssd
```

Use preferred node affinity. Scale the Deployment and observe placement.

```bash
kubectl get pods -o wide
kubectl describe pod <pod>
```

Goal: learn the difference between preferred and required rules.

### Scenario 4: Break Hard Anti-Affinity

Create a Deployment with 3 replicas and hard pod anti-affinity across hostname on a 2-node cluster.

Expected result: one Pod stays Pending.

Debug:

```bash
kubectl get pods -o wide
kubectl describe pod <pending-pod>
```

Goal: hard spreading rules need enough Nodes or topology domains.

### Scenario 5: Spread Replicas with Topology Constraints

Use `topologySpreadConstraints` with `ScheduleAnyway`, then switch to `DoNotSchedule`.

Observe:

```bash
kubectl get pods -o wide
kubectl describe pod <pod>
```

Goal: compare soft spreading with hard spreading.

### Scenario 6: Diagnose a Pending Pod

Given any Pending Pod:

```bash
kubectl describe pod <pod>
kubectl get nodes --show-labels
kubectl describe node <node>
kubectl get pvc
kubectl get events --sort-by=.lastTimestamp
```

Classify the root cause:

| Finding | Category |
| --- | --- |
| Untolerated taint | Taint/toleration |
| Node affinity mismatch | Label/affinity |
| Insufficient CPU or memory | Resources |
| Unbound PVC | Storage |
| Volume node affinity conflict | Storage topology |
| Too few topology domains | Anti-affinity/spread |

## Fast Exam Checklist

| Task | Command or field |
| --- | --- |
| Show Node labels. | `kubectl get nodes --show-labels` |
| Add Node label. | `kubectl label node <node> key=value` |
| Add Node taint. | `kubectl taint node <node> key=value:NoSchedule` |
| Remove Node taint. | `kubectl taint node <node> key=value:NoSchedule-` |
| Simple placement. | `spec.nodeSelector` |
| Allow tainted Node. | `spec.tolerations` |
| Force dedicated Node. | Taint + toleration + `nodeSelector` |
| Advanced Node rules. | `affinity.nodeAffinity` |
| Keep Pods together. | `podAffinity` |
| Keep Pods apart. | `podAntiAffinity` |
| Spread replicas. | `topologySpreadConstraints` |
| Debug Pending Pod. | `kubectl describe pod <pod>` |

## Common Mistakes

| Mistake | Result | Fix |
| --- | --- | --- |
| Adding toleration only and expecting Pod to choose the tainted Node. | Pod may schedule elsewhere. | Add `nodeSelector` or node affinity too. |
| Hard affinity label does not exist on any Node. | Pod stays Pending. | Label a Node or relax the rule. |
| Hard anti-affinity with too many replicas. | Some Pods stay Pending. | Use preferred anti-affinity or add Nodes. |
| Taint key/value/effect mismatch. | Pod does not tolerate the Node. | Match the taint exactly or use `operator: Exists`. |
| Using `kubectl top` only for scheduling debug. | Misses requested resource pressure. | Check `kubectl describe node` allocatable and allocated requests. |
| Ignoring PVC events. | Storage-caused Pending looks like scheduling failure. | Check PVC, PV, StorageClass, and Pod events. |
| Removing control-plane taints on real clusters. | Workloads may land on control-plane Nodes. | Prefer explicit tolerations for controlled cases. |

## Interview Questions

| Question | Strong answer |
| --- | --- |
| What does a taint do? | It repels Pods from a Node unless the Pod has a matching toleration. |
| Does a toleration force placement onto a Node? | No. It only permits scheduling onto a tainted Node. Use node affinity or `nodeSelector` to attract the Pod. |
| What is the fastest way to schedule a Pod onto labeled Nodes? | Use `nodeSelector` for simple exact label matches. |
| When should you use node affinity instead of `nodeSelector`? | When you need operators like `In`, `NotIn`, soft preferences, or more complex rules. |
| How do you spread Pods across Nodes? | Use topology spread constraints or pod anti-affinity with `kubernetes.io/hostname`. |
| Why is a Pod Pending? | Check events first: taints, affinity mismatch, insufficient resources, PVC binding, or topology constraints. |
| What does `IgnoredDuringExecution` mean? | The rule is checked during scheduling, but the Pod is not evicted if labels change afterward. |
