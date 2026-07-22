# Kubernetes Deployment Strategies Study Guide

Use this guide to understand common application deployment patterns, what Kubernetes supports directly, and how to practice each pattern.

## Strategy Overview

| Strategy | What it does | Kubernetes support | Best for | Tradeoff |
| --- | --- | --- | --- | --- |
| Rolling update | Gradually replaces old Pods with new Pods. | Native Deployment strategy. | Most normal stateless app releases. | Old and new versions run at the same time. |
| Recreate | Stops old Pods before starting new Pods. | Native Deployment strategy. | Apps that cannot run two versions at once. | Causes downtime. |
| Blue/green | Runs old and new versions side by side, then switches traffic. | Built from Deployments and Services. | Fast rollback and clean cutover. | Needs double capacity during release. |
| Canary | Sends a small amount of traffic to the new version first. | Built from separate Deployments, labels, ingress, mesh, or gateway. | Risk reduction for production releases. | Native Services do not do weighted traffic by themselves. |
| A/B test | Routes users to different versions based on headers, cookies, or user traits. | Usually needs Ingress, Gateway API, or service mesh. | Product experiments. | More routing complexity. |
| Shadow traffic | Mirrors real traffic to a new version without returning its response. | Usually needs ingress, proxy, gateway, or mesh. | Testing production behavior safely. | New version must not cause side effects. |
| Stateful rolling update | Updates StatefulSet Pods in stable order. | Native StatefulSet behavior. | Databases and ordered stateful systems. | Slower and more sensitive to readiness. |
| DaemonSet rolling update | Updates one Pod per matching Node. | Native DaemonSet behavior. | Node agents such as log collectors or CNI helpers. | Node-level failures can affect rollout. |

## Rolling Update

Rolling update is the default Deployment strategy. Kubernetes creates new Pods and removes old Pods gradually.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
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

Practice:

```bash
kubectl apply -f web.yaml
kubectl set image deployment/web web=nginx:1.27
kubectl rollout status deployment/web
kubectl rollout history deployment/web
kubectl rollout undo deployment/web
```

Study notes:

```text
maxSurge controls how many extra Pods can exist during rollout.
maxUnavailable controls how many desired Pods can be unavailable.
Readiness probes decide when new Pods can receive traffic.
```

## Recreate

Recreate stops the old version before starting the new version.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  strategy:
    type: Recreate
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
```

Practice:

```bash
kubectl apply -f web-recreate.yaml
kubectl set image deployment/web web=nginx:1.27
kubectl get pods -w
```

Use when:

```text
The app cannot tolerate mixed versions.
The app uses a lock or singleton resource.
Short downtime is acceptable.
```

## Blue/Green

Blue/green keeps two versions running. The Service selector decides which version receives traffic.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
      version: blue
  template:
    metadata:
      labels:
        app: web
        version: blue
    spec:
      containers:
        - name: web
          image: nginx:1.26
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
      version: green
  template:
    metadata:
      labels:
        app: web
        version: green
    spec:
      containers:
        - name: web
          image: nginx:1.27
---
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web
    version: blue
  ports:
    - port: 80
      targetPort: 80
```

Practice:

```bash
kubectl apply -f blue-green.yaml
kubectl get endpoints web
kubectl patch service web -p '{"spec":{"selector":{"app":"web","version":"green"}}}'
kubectl get endpoints web
kubectl patch service web -p '{"spec":{"selector":{"app":"web","version":"blue"}}}'
```

Study notes:

```text
Rollback is just switching the Service selector back.
Both versions must be healthy before cutover.
This pattern is simple and exam-friendly.
```

## Canary

Canary releases run a small number of new-version Pods alongside stable Pods. With a normal Kubernetes Service, traffic is balanced across all matching Pods, not weighted by version.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-stable
spec:
  replicas: 4
  selector:
    matchLabels:
      app: web
      track: stable
  template:
    metadata:
      labels:
        app: web
        track: stable
    spec:
      containers:
        - name: web
          image: nginx:1.26
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
      track: canary
  template:
    metadata:
      labels:
        app: web
        track: canary
    spec:
      containers:
        - name: web
          image: nginx:1.27
---
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
```

Practice:

```bash
kubectl apply -f canary.yaml
kubectl get pods -l app=web --show-labels
kubectl scale deployment web-canary --replicas=2
kubectl scale deployment web-stable --replicas=3
kubectl rollout status deployment/web-canary
```

Study notes:

```text
Native Service traffic is roughly distributed across selected Pods.
For exact percentages, use Ingress, Gateway API, or a service mesh.
Canary rollback can be scaling canary to zero.
```

## A/B Testing

A/B testing routes different users to different versions based on request properties.

| Routing signal | Example |
| --- | --- |
| Header | `X-Experiment: checkout-v2` |
| Cookie | `experiment=checkout-v2` |
| User group | beta users, internal users |
| Geography | country or region |

Kubernetes Services do not inspect HTTP headers. Use one of these:

```text
Ingress controller with header/cookie routing
Gateway API with HTTPRoute rules
Service mesh traffic policy
Application-level routing
```

Practice idea:

```bash
kubectl create deployment app-a --image=nginx
kubectl create deployment app-b --image=httpd
kubectl expose deployment app-a --port=80 --name=app-a
kubectl expose deployment app-b --port=80 --name=app-b
kubectl get svc
```

Then create Ingress or Gateway routing rules based on your local controller.

## Shadow Traffic

Shadow traffic sends a copy of real traffic to a new version but returns only the stable version response.

Use when:

```text
You want production-like testing.
The new version should not affect users.
You can prevent duplicate writes or side effects.
```

Common tools:

```text
Ingress controller traffic mirroring
Gateway or proxy traffic mirroring
Service mesh mirroring
Application-level event replay
```

Study notes:

```text
Shadow traffic is not a native Deployment strategy.
It is a routing/proxy capability.
The shadow version must be safe to receive duplicate requests.
```

## StatefulSet Rolling Updates

StatefulSets update Pods in a stable, ordered way. Pods have stable names like `db-0`, `db-1`, and `db-2`.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: db
spec:
  serviceName: db
  replicas: 3
  updateStrategy:
    type: RollingUpdate
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
        - name: db
          image: nginx:1.26
```

Practice:

```bash
kubectl apply -f statefulset.yaml
kubectl set image statefulset/db db=nginx:1.27
kubectl rollout status statefulset/db
kubectl get pods -w
```

Study notes:

```text
StatefulSets preserve identity and storage.
Readiness matters because ordered updates wait on Pod health.
```

## DaemonSet Rolling Updates

DaemonSets run one Pod on every matching Node and can be updated gradually.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-agent
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  selector:
    matchLabels:
      app: node-agent
  template:
    metadata:
      labels:
        app: node-agent
    spec:
      containers:
        - name: agent
          image: busybox
          command: ["sh", "-c", "sleep 3600"]
```

Practice:

```bash
kubectl apply -f daemonset.yaml
kubectl set image daemonset/node-agent agent=busybox:1.36
kubectl rollout status daemonset/node-agent
kubectl get pods -o wide -l app=node-agent
```

## Practice Scenarios

### Scenario 1: Fix a Broken Rolling Update

Task:

```text
A Deployment rollout is stuck. Find why, fix it, and complete the rollout.
```

Practice commands:

```bash
kubectl create deployment broken --image=nginx:badtag --replicas=3
kubectl rollout status deployment/broken
kubectl describe deployment broken
kubectl get pods
kubectl describe pod <pod>
kubectl set image deployment/broken nginx=nginx:1.27
kubectl rollout status deployment/broken
```

What to learn:

```text
ImagePullBackOff blocks rollout.
Events usually reveal the cause.
Rollout status is the fastest first signal.
```

### Scenario 2: Practice Blue/Green Cutover

Task:

```text
Deploy blue and green versions. Send traffic to blue, switch to green, then roll back.
```

Practice commands:

```bash
kubectl apply -f blue-green.yaml
kubectl get endpoints web
kubectl patch service web -p '{"spec":{"selector":{"app":"web","version":"green"}}}'
kubectl get endpoints web
kubectl patch service web -p '{"spec":{"selector":{"app":"web","version":"blue"}}}'
```

What to learn:

```text
The Service selector is the traffic switch.
Endpoint changes prove which Pods receive traffic.
```

### Scenario 3: Practice Canary Scaling

Task:

```text
Run 4 stable Pods and 1 canary Pod. Increase canary gradually.
```

Practice commands:

```bash
kubectl apply -f canary.yaml
kubectl get pods -l app=web --show-labels
kubectl scale deployment web-canary --replicas=2
kubectl scale deployment web-stable --replicas=3
kubectl get pods -l app=web
```

What to learn:

```text
Native Kubernetes canary by Pod count is approximate.
Weighted traffic needs another routing layer.
```

### Scenario 4: Compare Recreate and RollingUpdate

Task:

```text
Create one Deployment with Recreate and one with RollingUpdate. Watch Pod behavior during image updates.
```

Practice commands:

```bash
kubectl get pods -w
kubectl set image deployment/web web=nginx:1.27
kubectl describe deployment web
```

What to learn:

```text
Recreate causes downtime.
RollingUpdate overlaps old and new Pods.
```

## Exam and Interview Tips

| Question | Good answer |
| --- | --- |
| Which strategy is default for Deployment? | RollingUpdate. |
| What does Recreate do? | Deletes old Pods before creating new Pods. |
| How do you roll back a Deployment? | `kubectl rollout undo deployment/<name>`. |
| What controls rolling update speed? | `maxSurge`, `maxUnavailable`, readiness, and replica count. |
| Is blue/green native? | It is built with Deployments and Service selectors. |
| Is weighted canary native to Services? | No. Use Ingress, Gateway API, service mesh, or another router for exact weights. |
| Why are readiness probes important? | They prevent unready Pods from receiving Service traffic. |
| How do you verify traffic targets? | Check Service selectors and Endpoints or EndpointSlices. |
