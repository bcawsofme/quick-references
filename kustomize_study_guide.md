# Kustomize Study Guide

Use this guide to understand how Kustomize builds Kubernetes manifests from reusable bases and environment-specific overlays.

## Mental Model

| Term | Meaning | Study note |
| --- | --- | --- |
| Kustomize | A Kubernetes-native manifest customization tool. | Built into `kubectl` as `kubectl kustomize` and `kubectl apply -k`. |
| kustomization.yaml | The file that declares resources, patches, generators, and transforms. | Each base or overlay has one. |
| Base | Shared reusable manifests. | Usually environment-neutral. |
| Overlay | Environment-specific customization of a base. | Common overlays: `dev`, `stage`, `prod`. |
| Resource | A manifest included in a kustomization. | Deployment, Service, ConfigMap, etc. |
| Patch | A partial change applied to one or more resources. | Use for image, replicas, env, labels, etc. |
| Generator | Creates ConfigMaps or Secrets from literals, files, or env files. | Hash suffix changes when data changes. |
| Transformer | Applies changes such as labels, annotations, namespace, name prefix, or image tags. | Often affects many resources at once. |

## Common Directory Layout

```text
app/
  base/
    kustomization.yaml
    deployment.yaml
    service.yaml
  overlays/
    dev/
      kustomization.yaml
      patch-replicas.yaml
    prod/
      kustomization.yaml
      patch-replicas.yaml
```

Build and apply:

```bash
kubectl kustomize app/overlays/dev
kubectl apply -k app/overlays/dev
kubectl diff -k app/overlays/dev
kubectl delete -k app/overlays/dev
```

## Base Example

`base/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
commonLabels:
  app.kubernetes.io/name: web
```

`base/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 1
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
          image: nginx:1.26
          ports:
            - containerPort: 80
```

## Overlay Example

`overlays/prod/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
namespace: prod
namePrefix: prod-
images:
  - name: nginx
    newTag: "1.27"
patches:
  - path: patch-replicas.yaml
    target:
      kind: Deployment
      name: web
```

`overlays/prod/patch-replicas.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
```

## Images

```yaml
images:
  - name: nginx
    newName: registry.example.com/platform/nginx
    newTag: "1.27"
```

Commands:

```bash
kustomize edit set image nginx=nginx:1.27
kubectl kustomize overlays/prod
```

Study notes:

```text
Image transforms avoid patching full container specs.
They match by image name.
```

## ConfigMap and Secret Generators

```yaml
configMapGenerator:
  - name: app-config
    literals:
      - MODE=prod
      - LOG_LEVEL=info

secretGenerator:
  - name: app-secret
    literals:
      - TOKEN=example
```

Use generated config in a Deployment:

```yaml
envFrom:
  - configMapRef:
      name: app-config
  - secretRef:
      name: app-secret
```

Study notes:

```text
Generated names usually include a hash suffix.
The hash helps trigger rollouts when config changes.
Use generatorOptions only when you understand the rollout tradeoff.
```

## Patches

Strategic merge patch:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  template:
    spec:
      containers:
        - name: web
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
```

JSON 6902 patch:

```yaml
patches:
  - target:
      kind: Deployment
      name: web
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 3
```

## Labels and Annotations

```yaml
commonLabels:
  app.kubernetes.io/part-of: platform

commonAnnotations:
  owner: platform-team
```

Study notes:

```text
Be careful with labels that affect selectors.
Changing selector labels can break Services and Deployments.
```

## Names and Namespaces

```yaml
namespace: dev
namePrefix: dev-
nameSuffix: -v1
```

Use when:

```text
Multiple environments share a cluster.
You need predictable environment-specific names.
You want one base reused across several overlays.
```

## Components

Components are optional reusable chunks that can be included by overlays.

```yaml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component
resources:
  - monitoring-sidecar.yaml
```

Overlay:

```yaml
components:
  - ../../components/monitoring
```

Study notes:

```text
Use components for optional features.
Use bases for required shared resources.
```

## Debugging Workflow

```bash
kubectl kustomize overlays/dev
kubectl diff -k overlays/dev
kubectl apply -k overlays/dev --dry-run=server
kubectl apply -k overlays/dev
kubectl get all -n dev
```

| Symptom | Check |
| --- | --- |
| Resource missing. | `resources` paths and build output. |
| Patch not applied. | Patch target kind, name, namespace, and apiVersion. |
| Image not changed. | Image name match. |
| Service has no endpoints. | Labels and selectors after build. |
| Config change did not roll Pods. | Generated ConfigMap/Secret name and Deployment references. |

## Kustomize vs Helm

| Topic | Kustomize | Helm |
| --- | --- | --- |
| Style | Patch and transform plain YAML. | Template YAML with values. |
| Package model | No release object. | Release object and history. |
| Rollback | Use Git or previous manifests. | `helm rollback`. |
| Best for | Environment overlays and GitOps. | App packaging and configurable installs. |
| Language | YAML patches and transformers. | Go templates. |

## Practice Scenarios

### Scenario 1: Build a Base and Two Overlays

Task:

```text
Create a base Deployment and Service. Create dev and prod overlays with different replica counts and image tags.
```

Commands:

```bash
kubectl kustomize overlays/dev
kubectl kustomize overlays/prod
kubectl diff -k overlays/dev
kubectl apply -k overlays/dev
```

### Scenario 2: Patch Resources

Task:

```text
Add CPU and memory requests only in the prod overlay.
```

Practice:

```bash
kubectl kustomize overlays/prod | grep -A6 resources
kubectl apply -k overlays/prod
kubectl describe pod <pod>
```

### Scenario 3: Generate Config

Task:

```text
Generate a ConfigMap from literals and mount it into a Deployment.
```

Practice:

```bash
kubectl kustomize overlays/dev
kubectl apply -k overlays/dev
kubectl get configmap
kubectl describe deployment web
```

### Scenario 4: Debug a Broken Patch

Task:

```text
Create a patch with the wrong target name. Build the overlay, find why it failed, and fix it.
```

Practice:

```bash
kubectl kustomize overlays/dev
kubectl diff -k overlays/dev
```

## Exam and Interview Tips

| Question | Good answer |
| --- | --- |
| What is a base? | Shared manifests reused by overlays. |
| What is an overlay? | Environment-specific changes layered on top of a base. |
| How do you preview output? | `kubectl kustomize <dir>`. |
| How do you apply output? | `kubectl apply -k <dir>`. |
| What is a generator hash for? | It changes generated names when config changes, helping roll workloads. |
| What commonly breaks overlays? | Wrong paths, wrong patch target names, selector label changes, and image name mismatches. |
