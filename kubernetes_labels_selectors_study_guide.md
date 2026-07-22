# Kubernetes Labels and Selectors Study Guide

Use this guide to study how Kubernetes uses labels, selectors, and annotations to organize, find, route to, schedule, and manage resources. This is a high-value CKA and CKAD topic because labels connect many different objects together.

## Quick Visual

![Kubernetes workload and service flow](images/kubernetes-workload-service.svg)

```text
Deployment selector -> ReplicaSet selector -> Pod template labels -> Pods
Service selector    -> matching Pod labels -> EndpointSlice -> Pod IPs
NetworkPolicy       -> podSelector / namespaceSelector -> allowed traffic
Scheduler           -> node labels + Pod constraints -> chosen Node
```

## Core Definitions

| Term | Meaning | Exam note |
| --- | --- | --- |
| Label | Key-value metadata attached to an object. | Used for grouping and selecting objects. |
| Selector | A query that matches labels. | Wrong selectors commonly break Services, Deployments, and NetworkPolicies. |
| Annotation | Key-value metadata for tools, humans, or controllers. | Not used for normal object selection. |
| Equality selector | Matches exact label values. | Example: `app=web`, `env!=prod`. |
| Set-based selector | Matches labels against a set of values. | Example: `env in (dev,qa)`, `tier notin (cache)`. |
| Pod template labels | Labels placed under `spec.template.metadata.labels`. | These labels are copied onto Pods created by a Deployment, Job, or StatefulSet. |
| `matchLabels` | A selector map requiring exact key-value matches. | Common in Deployments and NetworkPolicies. |
| `matchExpressions` | A selector list with operators such as `In`, `NotIn`, `Exists`, and `DoesNotExist`. | More flexible than `matchLabels`. |
| OwnerReference | Metadata that links dependent resources to their owner. | Labels select; owner references control garbage collection. |

## Labels vs Annotations

| Use case | Use labels | Use annotations |
| --- | --- | --- |
| Select Pods for a Service | Yes | No |
| Select Pods for a NetworkPolicy | Yes | No |
| Find resources with `kubectl get -l` | Yes | No |
| Add ownership/team metadata | Often | Often |
| Store tool-specific config | Usually no | Yes |
| Store long or unstructured metadata | No | Yes |
| Drive scheduling with Node labels | Yes | No |

Good label examples:

```yaml
metadata:
  labels:
    app: web
    tier: frontend
    env: dev
    owner: platform
```

Good annotation examples:

```yaml
metadata:
  annotations:
    runbook: "https://example.com/runbooks/web"
    description: "frontend web workload"
```

## Label Commands

```bash
# Show labels on resources
kubectl get pods --show-labels
kubectl get deploy --show-labels
kubectl get nodes --show-labels

# Add or update labels
kubectl label pod nginx app=web
kubectl label deployment web tier=frontend
kubectl label deployment web tier=edge --overwrite

# Remove labels
kubectl label pod nginx app-
kubectl label deployment web tier-

# Add or remove annotations
kubectl annotate pod nginx owner=platform
kubectl annotate pod nginx owner-
kubectl annotate deployment web runbook=https://example.com/runbook --overwrite
```

## Selecting Resources

```bash
# Equality selectors
kubectl get pods -l app=web
kubectl get pods -l app!=web
kubectl get pods -l app=web,tier=frontend

# Set-based selectors
kubectl get pods -l 'env in (dev,qa)'
kubectl get pods -l 'env notin (prod,stage)'
kubectl get pods -l 'owner'
kubectl get pods -l '!owner'

# Use selectors with other commands
kubectl describe pods -l app=web
kubectl logs -l app=web --tail=50
kubectl delete pods -l app=web
kubectl rollout status deployment -l app=web
```

Be careful with bulk delete commands. In an exam, run `kubectl get ... -l ...` first to confirm the selector matches only the intended resources.

## Deployment Management

A Deployment manages Pods through selectors and Pod template labels.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  labels:
    app: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
```

Important rules:

| Rule | Why it matters |
| --- | --- |
| `spec.selector.matchLabels` must match `spec.template.metadata.labels`. | Otherwise the Deployment cannot manage its Pods correctly. |
| Deployment selectors are immutable after creation. | If the selector is wrong, recreate the Deployment or use a new one. |
| Extra Pod labels are fine. | The selector only needs to match the required labels. |
| Labels are not the same as owner references. | The Deployment selects Pods by labels, but Kubernetes tracks ownership through owner references. |

Check the relationship:

```bash
kubectl get deploy web -o yaml
kubectl get rs -l app=web
kubectl get pods -l app=web --show-labels
kubectl describe deploy web
```

## Service Management

A Service routes traffic to Pods selected by labels. The selected ready Pods become EndpointSlice addresses.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web
    tier: frontend
  ports:
  - port: 80
    targetPort: 80
```

Troubleshooting path:

```bash
kubectl get svc web -o yaml
kubectl get pods --show-labels
kubectl get pods -l app=web,tier=frontend
kubectl get endpointslice -l kubernetes.io/service-name=web
kubectl describe svc web
```

Common failure:

| Symptom | Likely cause | Check |
| --- | --- | --- |
| Service has no endpoints. | Service selector does not match Pod labels. | `kubectl get pods --show-labels` |
| Service routes to wrong Pods. | Selector is too broad. | `kubectl get pods -l <selector>` |
| Ingress returns 503. | Backend Service has no ready endpoints. | Service selector, Pod readiness, EndpointSlices. |
| DNS resolves but traffic fails. | DNS is fine; Service backend selection is broken. | `kubectl get endpointslice`. |

Patch a Service selector:

```bash
kubectl patch service web -p '{"spec":{"selector":{"app":"web","tier":"frontend"}}}'
```

## NetworkPolicy Selectors

NetworkPolicies use labels to select target Pods and allowed traffic sources.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-from-web
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: web
    ports:
    - protocol: TCP
      port: 8080
```

Selector meanings:

| Selector | Meaning |
| --- | --- |
| `spec.podSelector` | Selects the Pods the policy applies to. |
| `ingress.from[].podSelector` | Selects allowed source Pods in the same namespace. |
| `ingress.from[].namespaceSelector` | Selects allowed source namespaces by namespace labels. |
| Empty `podSelector: {}` | Selects all Pods in the namespace. |

Exam warning: a NetworkPolicy only works if the cluster CNI enforces NetworkPolicy.

## Node Labels and Scheduling

Nodes also have labels. Pods can use those labels for placement.

```bash
kubectl get nodes --show-labels
kubectl label node worker01 disk=ssd
kubectl label node worker01 workload=apps
```

Simple `nodeSelector`:

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

Check scheduling:

```bash
kubectl get pod nginx-ssd -o wide
kubectl describe pod nginx-ssd
kubectl describe node worker01
```

If a Pod is Pending, check whether any Node has the required label.

## Kustomize Labels

Kustomize can apply labels across many resources.

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- deployment.yaml
- service.yaml
commonLabels:
  app.kubernetes.io/name: web
  app.kubernetes.io/part-of: storefront
```

Be careful: broad label transformers can affect selectors depending on Kustomize behavior and configuration. Always inspect the rendered YAML.

```bash
kubectl kustomize overlays/dev
kubectl kustomize overlays/dev | kubectl apply --dry-run=server -f -
```

## Recommended Label Keys

Kubernetes commonly uses the `app.kubernetes.io/*` label convention.

| Label | Example | Purpose |
| --- | --- | --- |
| `app.kubernetes.io/name` | `web` | Application name. |
| `app.kubernetes.io/instance` | `web-dev` | Unique instance of the app. |
| `app.kubernetes.io/version` | `1.2.3` | App version. |
| `app.kubernetes.io/component` | `frontend` | Component role. |
| `app.kubernetes.io/part-of` | `storefront` | Larger system name. |
| `app.kubernetes.io/managed-by` | `Helm` | Tool managing the object. |

For fast exam work, simple keys such as `app`, `tier`, `env`, and `role` are usually enough unless the task gives a specific convention.

## Practice Scenarios

### Scenario 1: Find and Label Pods

Create two Pods, label only one as `app=web`, and select it.

```bash
kubectl run web --image=nginx
kubectl run api --image=nginx
kubectl label pod web app=web tier=frontend
kubectl get pods -l app=web --show-labels
```

Goal: only the `web` Pod should match.

### Scenario 2: Fix a Service with No Endpoints

Create a Pod and a Service with mismatched labels. Then fix the Service selector.

```bash
kubectl run web --image=nginx --labels app=web
kubectl expose pod web --port=80 --name=web
kubectl patch service web -p '{"spec":{"selector":{"app":"wrong"}}}'

kubectl get endpointslice -l kubernetes.io/service-name=web
kubectl get pods --show-labels
kubectl patch service web -p '{"spec":{"selector":{"app":"web"}}}'
kubectl get endpointslice -l kubernetes.io/service-name=web
```

Goal: EndpointSlice should be empty before the fix and populated after the fix.

### Scenario 3: Repair a Deployment Selector

Create a Deployment YAML where `spec.selector.matchLabels` does not match `spec.template.metadata.labels`. Try a dry run, then fix it.

```bash
kubectl create deployment web --image=nginx --dry-run=client -o yaml > web.yaml
```

Edit the labels intentionally, then validate:

```bash
kubectl apply --dry-run=server -f web.yaml
```

Goal: understand that Deployment selectors and Pod template labels must align.

### Scenario 4: Select Logs by Label

Create a Deployment and fetch logs from all matching Pods.

```bash
kubectl create deployment web --image=nginx --replicas=3
kubectl get pods -l app=web
kubectl logs -l app=web --tail=20
```

Goal: use labels for operational commands instead of copying Pod names.

### Scenario 5: Schedule by Node Label

Label a worker Node and schedule a Pod onto it.

```bash
kubectl get nodes
kubectl label node <node> workload=practice
```

Create:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: scheduled-by-label
spec:
  nodeSelector:
    workload: practice
  containers:
  - name: nginx
    image: nginx:1.27
```

Check:

```bash
kubectl apply -f pod.yaml
kubectl get pod scheduled-by-label -o wide
```

Goal: the Pod lands on the labeled Node.

### Scenario 6: NetworkPolicy Targeting

Create labels for `app=web` and `app=api`, then apply a policy allowing web to reach api.

```bash
kubectl get pods --show-labels
kubectl label pod web app=web --overwrite
kubectl label pod api app=api --overwrite
```

Goal: read the NetworkPolicy selectors and identify which Pods are protected and which Pods are allowed.

## Fast Exam Checklist

| Task | Command or check |
| --- | --- |
| Show labels. | `kubectl get pods --show-labels` |
| Select resources. | `kubectl get pods -l app=web` |
| Add label. | `kubectl label pod nginx app=web` |
| Overwrite label. | `kubectl label pod nginx app=api --overwrite` |
| Remove label. | `kubectl label pod nginx app-` |
| Check Service targets. | `kubectl get endpointslice -l kubernetes.io/service-name=<svc>` |
| Debug no endpoints. | Compare Service selector with Pod labels. |
| Debug Pending Pod with `nodeSelector`. | Compare Pod `nodeSelector` with Node labels. |
| Debug NetworkPolicy. | Check `podSelector`, `namespaceSelector`, and Pod labels. |
| Avoid accidental bulk action. | Run `kubectl get ... -l ...` before delete or patch. |

## Common Mistakes

| Mistake | Result | Fix |
| --- | --- | --- |
| Service selector does not match Pod labels. | Service has no endpoints. | Patch Service selector or Pod labels. |
| Deployment selector does not match Pod template labels. | Deployment creation fails or cannot manage Pods correctly. | Align `spec.selector.matchLabels` with `spec.template.metadata.labels`. |
| Selector is too broad. | Service, logs, or delete command affects extra Pods. | Add another label condition. |
| Annotation used where label is required. | Selector does not match anything. | Put searchable metadata in labels. |
| Node label is missing. | Pod with `nodeSelector` stays Pending. | Label a suitable Node or change the selector. |
| Kustomize changes selector labels unexpectedly. | Services or Deployments break after render. | Inspect `kubectl kustomize` output before applying. |
