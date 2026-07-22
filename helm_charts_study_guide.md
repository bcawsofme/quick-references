# Helm Charts Study Guide

Use this guide to understand how Helm charts are structured, how templates render into Kubernetes manifests, and how to practice chart authoring and debugging.

## Mental Model

| Term | Meaning | Study note |
| --- | --- | --- |
| Helm | Kubernetes package manager. | Installs, upgrades, rolls back, and uninstalls releases. |
| Chart | A package of Kubernetes templates, values, metadata, and optional dependencies. | A chart is the thing you install. |
| Release | A deployed instance of a chart. | One chart can have many releases. |
| Template | A Kubernetes manifest with Go template expressions. | Rendered into plain YAML before applying. |
| Values | Input data used by templates. | Defaults live in `values.yaml`; override with `-f` or `--set`. |
| Chart.yaml | Chart metadata file. | Contains chart name, version, app version, and dependencies. |
| values.yaml | Default configuration for the chart. | Should be readable and safe by default. |
| templates/ | Directory containing templated Kubernetes manifests. | Deployment, Service, Ingress, ConfigMap, etc. |
| _helpers.tpl | Template helper definitions. | Common place for names, labels, and reusable snippets. |
| Dependency | Another chart used by this chart. | Declared in `Chart.yaml` and pulled into `charts/`. |

## Chart Structure

```text
my-chart/
  Chart.yaml
  values.yaml
  templates/
    _helpers.tpl
    deployment.yaml
    service.yaml
    ingress.yaml
    configmap.yaml
    secret.yaml
    tests/
      test-connection.yaml
```

Create and inspect:

```bash
helm create my-chart
find my-chart -maxdepth 3 -type f
helm lint my-chart
helm template demo my-chart
```

## Chart.yaml

```yaml
apiVersion: v2
name: my-chart
description: A sample application chart
type: application
version: 0.1.0
appVersion: "1.0.0"
```

| Field | Meaning |
| --- | --- |
| `apiVersion` | Helm chart API version. Use `v2` for Helm 3 charts. |
| `name` | Chart name. |
| `type` | `application` or `library`. |
| `version` | Chart package version. |
| `appVersion` | Version of the app being deployed. |

## values.yaml

```yaml
replicaCount: 2

image:
  repository: nginx
  tag: "1.27"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi
```

Override values:

```bash
helm template demo my-chart -f values-dev.yaml
helm template demo my-chart --set image.tag=1.28
helm upgrade --install demo my-chart -f values-prod.yaml
```

## Template Basics

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-chart.fullname" . }}
  labels:
    {{- include "my-chart.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "my-chart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "my-chart.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
```

## Common Template Objects

| Object | Meaning |
| --- | --- |
| `.Values` | Values from `values.yaml`, `-f`, and `--set`. |
| `.Chart` | Chart metadata from `Chart.yaml`. |
| `.Release` | Release metadata such as name, namespace, and revision. |
| `.Capabilities` | Cluster capabilities discovered by Helm. |
| `.Files` | Access to non-template files packaged in the chart. |

## Useful Template Functions

| Function | Use |
| --- | --- |
| `default` | Provide fallback values. |
| `quote` | Quote a value. |
| `required` | Fail rendering if a value is missing. |
| `include` | Render a named template. |
| `tpl` | Render a string as a template. |
| `toYaml` | Convert data to YAML. |
| `nindent` | Add newline and indentation. |

Example:

```yaml
imagePullPolicy: {{ .Values.image.pullPolicy | default "IfNotPresent" | quote }}
```

## Conditionals and Loops

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "my-chart.fullname" . }}
{{- end }}
```

```yaml
env:
{{- range .Values.env }}
  - name: {{ .name }}
    value: {{ .value | quote }}
{{- end }}
```

## Helpers

`templates/_helpers.tpl`:

```gotemplate
{{- define "my-chart.fullname" -}}
{{- printf "%s-%s" .Release.Name .Chart.Name | trunc 63 | trimSuffix "-" -}}
{{- end -}}

{{- define "my-chart.labels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end -}}
```

Study notes:

```text
Helpers keep labels and names consistent.
Selector labels should be stable across upgrades.
Changing selector labels can break or replace workloads.
```

## Dependencies

`Chart.yaml`:

```yaml
dependencies:
  - name: redis
    version: "19.x.x"
    repository: "https://charts.bitnami.com/bitnami"
```

Commands:

```bash
helm dependency list my-chart
helm dependency update my-chart
helm dependency build my-chart
```

## Install, Upgrade, and Rollback Practice

```bash
helm install demo ./my-chart
helm status demo
helm upgrade demo ./my-chart --set image.tag=1.28
helm history demo
helm rollback demo 1
helm uninstall demo
```

## Debugging Workflow

```bash
helm lint ./my-chart
helm template demo ./my-chart --debug
helm install demo ./my-chart --dry-run --debug
helm get values demo
helm get manifest demo
helm get all demo
kubectl describe pod <pod>
kubectl logs <pod>
```

| Symptom | Check |
| --- | --- |
| YAML render fails. | Template syntax, missing values, indentation. |
| Install succeeds but Pods fail. | Rendered manifest, Pod events, logs. |
| Upgrade changes too much. | `helm diff` plugin or compare `helm template` output. |
| Service has no endpoints. | Labels and selectors. |
| Rollback does not fix app. | ConfigMaps, Secrets, PVCs, and external state. |

## Practice Scenarios

### Scenario 1: Create a Simple App Chart

Task:

```text
Create a chart that deploys nginx with configurable replica count, image tag, and service port.
```

Commands:

```bash
helm create web
helm template demo web
helm install demo web --set replicaCount=2 --set image.tag=1.27
kubectl get deploy,svc,pods
```

### Scenario 2: Add ConfigMap Values

Task:

```text
Add a ConfigMap template that renders key-value pairs from `.Values.config`.
```

Values:

```yaml
config:
  MODE: practice
  LOG_LEVEL: debug
```

Template pattern:

```yaml
data:
{{- range $key, $value := .Values.config }}
  {{ $key }}: {{ $value | quote }}
{{- end }}
```

### Scenario 3: Debug Bad Indentation

Task:

```text
Break a template by removing indentation. Use Helm commands to find the render error.
```

Commands:

```bash
helm lint web
helm template demo web --debug
```

### Scenario 4: Practice Rollback

Task:

```text
Install a chart, upgrade it to a bad image, observe the failure, then roll back.
```

Commands:

```bash
helm install demo web --set image.tag=1.27
helm upgrade demo web --set image.tag=badtag
kubectl get pods
helm history demo
helm rollback demo 1
kubectl rollout status deployment/demo-web
```

## Exam and Interview Tips

| Question | Good answer |
| --- | --- |
| What does Helm render? | Kubernetes manifests. |
| What is a release? | An installed instance of a chart. |
| How do you preview manifests? | `helm template` or `helm install --dry-run --debug`. |
| How do you override values? | `-f values.yaml`, repeated `-f`, or `--set`. |
| How do you roll back? | `helm rollback <release> <revision>`. |
| What commonly breaks charts? | Bad indentation, missing values, selector changes, and incorrect helper templates. |
