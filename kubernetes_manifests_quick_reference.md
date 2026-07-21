# Kubernetes Manifests Quick Reference Guide

## Pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - name: nginx
      image: nginx
      ports:
        - containerPort: 80
```

## Deployment
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
      containers:
        - name: nginx
          image: nginx
          ports:
            - containerPort: 80
```

## Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  type: ClusterIP
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
```

## ConfigMap and Secret Usage
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  MODE: prod
---
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
stringData:
  TOKEN: example
```

## Environment from ConfigMap and Secret
```yaml
env:
  - name: MODE
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: MODE
  - name: TOKEN
    valueFrom:
      secretKeyRef:
        name: app-secret
        key: TOKEN
```

## Probes
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

## Resources
```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi
```

## Volumes
```yaml
volumes:
  - name: data
    emptyDir: {}
containers:
  - name: app
    image: busybox
    volumeMounts:
      - name: data
        mountPath: /data
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
  resources:
    requests:
      storage: 1Gi
```

## Job
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: oneoff
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: task
          image: busybox
          command: ["sh", "-c", "date"]
```

## CronJob
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup
spec:
  schedule: "*/5 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: task
              image: busybox
              command: ["sh", "-c", "date"]
```

## Ingress
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
spec:
  rules:
    - host: web.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web
                port:
                  number: 80
```
