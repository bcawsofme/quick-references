# Kubernetes DNS and Ingress Study Guide

Use this guide to understand how in-cluster DNS, Services, Endpoints, and Ingress fit together, and how to practice common routing and troubleshooting scenarios.

## Mental Model

```text
Client -> DNS name -> Ingress controller -> Ingress rule -> Service -> EndpointSlice -> Pod

Pod -> CoreDNS -> Service DNS name -> Service ClusterIP -> EndpointSlice -> Pod
```

| Term | Meaning | Study note |
| --- | --- | --- |
| CoreDNS | Kubernetes DNS server. | Resolves Service names and other cluster DNS records. |
| Service | Stable virtual IP and DNS name for a set of Pods. | DNS usually resolves Services, not individual Pods. |
| EndpointSlice | Object listing backend Pod IPs for a Service. | If this is empty, Service traffic has nowhere to go. |
| ClusterIP | Internal virtual IP for a Service. | Default Service type. |
| Headless Service | Service with `clusterIP: None`. | DNS returns backend Pod IPs directly. |
| Ingress | HTTP/HTTPS routing rules to Services. | Requires an Ingress controller. |
| Ingress controller | The actual proxy that implements Ingress rules. | NGINX, Traefik, HAProxy, cloud controllers, etc. |
| IngressClass | Selects which Ingress controller should handle an Ingress. | Important when a cluster has multiple controllers. |
| TLS secret | Kubernetes Secret containing certificate and key. | Used by Ingress for HTTPS. |

## Kubernetes DNS Names

| Name | Resolves from | Example |
| --- | --- | --- |
| Short Service name | Same namespace | `web` |
| Service plus namespace | Any namespace | `web.default` |
| Full Service name | Any namespace | `web.default.svc.cluster.local` |
| Headless Service record | Any namespace | `db.default.svc.cluster.local` |
| Pod DNS record | Depends on cluster config | Less commonly used than Service DNS. |

Practice:

```bash
kubectl create namespace dns-lab
kubectl create deployment web --image=nginx -n dns-lab
kubectl expose deployment web --port=80 -n dns-lab
kubectl run dns-test --image=busybox --rm -it --restart=Never -- nslookup web.dns-lab
kubectl run dns-test --image=busybox --rm -it --restart=Never -- wget -qO- http://web.dns-lab
```

## CoreDNS Checks

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl get svc -n kube-system kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns
kubectl describe configmap coredns -n kube-system
kubectl rollout status deployment/coredns -n kube-system
```

| Symptom | Check |
| --- | --- |
| DNS lookup fails for all Services. | CoreDNS Pods, kube-dns Service, CoreDNS logs. |
| DNS works in one namespace but not another. | NetworkPolicy, Pod DNS config, namespace typo. |
| Short name fails from another namespace. | Use `<service>.<namespace>`. |
| DNS resolves but connection fails. | Service, EndpointSlice, Pod readiness, port mismatch. |

## Service to Endpoint Flow

```text
Service selector -> matching Pod labels -> EndpointSlice addresses -> traffic target
```

Service:

```yaml
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

Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 2
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

Verify:

```bash
kubectl get svc web
kubectl get endpoints web
kubectl get endpointslice -l kubernetes.io/service-name=web
kubectl get pods -l app=web -o wide
kubectl describe svc web
```

## Service Types and DNS

| Service type | DNS behavior | External access |
| --- | --- | --- |
| ClusterIP | Gets cluster DNS and internal virtual IP. | Internal only. |
| NodePort | Gets cluster DNS and opens a port on each Node. | `<node-ip>:<node-port>`. |
| LoadBalancer | Gets cluster DNS and external load balancer if supported. | Cloud or load balancer integration. |
| ExternalName | DNS CNAME to an external name. | No selector, no endpoints. |
| Headless | DNS returns endpoint IPs instead of ClusterIP. | Useful for StatefulSets and direct Pod discovery. |

## Headless Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: db
spec:
  clusterIP: None
  selector:
    app: db
  ports:
    - port: 5432
      targetPort: 5432
```

Study notes:

```text
Headless Services do not get a ClusterIP.
DNS returns backend endpoints directly.
StatefulSets often use headless Services for stable network identity.
```

## ExternalName Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-api
spec:
  type: ExternalName
  externalName: api.example.com
```

Practice:

```bash
kubectl apply -f externalname.yaml
kubectl run dns-test --image=busybox --rm -it --restart=Never -- nslookup external-api.default
```

## Ingress Basics

Ingress defines HTTP routing. It does not route traffic by itself; an Ingress controller must be installed.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
spec:
  ingressClassName: nginx
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

Verify:

```bash
kubectl get ingress
kubectl describe ingress web
kubectl get ingressclass
kubectl get svc -A | grep -i ingress
kubectl get pods -A | grep -i ingress
```

## Ingress Path Types

| Path type | Meaning | Example |
| --- | --- | --- |
| `Exact` | Must match exactly. | `/login` matches `/login` only. |
| `Prefix` | Matches URL path prefix by path element. | `/api` matches `/api/v1`. |
| `ImplementationSpecific` | Controller-specific matching. | Behavior depends on controller. |

Example with multiple paths:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: apps
spec:
  ingressClassName: nginx
  rules:
    - host: apps.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api
                port:
                  number: 80
          - path: /web
            pathType: Prefix
            backend:
              service:
                name: web
                port:
                  number: 80
```

## Ingress TLS

TLS Secret:

```bash
kubectl create secret tls web-tls --cert=tls.crt --key=tls.key
```

Ingress:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - web.example.com
      secretName: web-tls
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

Check:

```bash
kubectl get secret web-tls
kubectl describe ingress web
openssl s_client -connect web.example.com:443 -servername web.example.com
curl -vk https://web.example.com
```

## Local Practice with Minikube

```bash
minikube start -p ingress-lab
minikube addons enable ingress -p ingress-lab
kubectl create deployment web --image=nginx
kubectl expose deployment web --port=80
kubectl apply -f ingress.yaml
kubectl get ingress
minikube ip -p ingress-lab
curl -H "Host: web.example.com" http://$(minikube ip -p ingress-lab)
```

Study notes:

```text
For local practice, you can use a Host header instead of public DNS.
In real environments, external DNS must point to the ingress controller load balancer.
```

## Troubleshooting DNS

```bash
kubectl run dns-test --image=busybox --rm -it --restart=Never -- nslookup kubernetes.default
kubectl run dns-test --image=busybox --rm -it --restart=Never -- nslookup web.default
kubectl run dns-test --image=busybox --rm -it --restart=Never -- cat /etc/resolv.conf
kubectl logs -n kube-system -l k8s-app=kube-dns
kubectl get events -A --sort-by=.metadata.creationTimestamp
```

| Problem | Likely cause | Check |
| --- | --- | --- |
| `nslookup web` fails. | Wrong namespace or Service missing. | `kubectl get svc -A`. |
| Full DNS name fails. | CoreDNS issue or network policy. | CoreDNS Pods/logs and policies. |
| DNS resolves but HTTP fails. | Service has no endpoints or wrong port. | `kubectl get endpoints,endpointslice`. |
| Works from one Pod but not another. | NetworkPolicy or Pod DNS config. | `kubectl describe pod` and policies. |

## Troubleshooting Ingress

```bash
kubectl get ingress -A
kubectl describe ingress <ingress>
kubectl get ingressclass
kubectl get svc -A | grep -i ingress
kubectl get pods -A | grep -i ingress
kubectl logs -n <namespace> <ingress-controller-pod>
kubectl get svc <service>
kubectl get endpoints <service>
curl -H "Host: <host>" http://<ingress-ip>
curl -vk https://<host>
```

| Problem | Likely cause | Check |
| --- | --- | --- |
| Ingress has no address. | Controller not installed or not watching class. | Ingress controller Pods, Service, IngressClass. |
| 404 from ingress controller. | Host/path rule mismatch. | `kubectl describe ingress`, curl Host header. |
| 502 or 503. | Backend Service has no ready endpoints. | Service selector, Endpoints, Pod readiness. |
| TLS error. | Secret missing, wrong cert, host mismatch. | TLS Secret and `openssl s_client`. |
| Works by Service but not Ingress. | Ingress rule/controller issue. | Ingress logs and class. |
| Works by Ingress IP but not DNS name. | External DNS issue. | `dig`, DNS records, load balancer address. |

## Practice Scenarios

### Scenario 1: Service DNS Lookup

Task:

```text
Create a Deployment and Service. Resolve it by short name, namespace name, and full cluster DNS name.
```

Commands:

```bash
kubectl create namespace dns-lab
kubectl create deployment web --image=nginx -n dns-lab
kubectl expose deployment web --port=80 -n dns-lab
kubectl run dns-test --image=busybox --rm -it --restart=Never -- nslookup web.dns-lab
kubectl run dns-test --image=busybox --rm -it --restart=Never -- nslookup web.dns-lab.svc.cluster.local
```

What to learn:

```text
Short names depend on namespace.
Full names work across namespaces.
```

### Scenario 2: Fix a Service with No Endpoints

Task:

```text
Create a Service with a selector that does not match Pods. Find and fix the issue.
```

Commands:

```bash
kubectl get svc web
kubectl get endpoints web
kubectl get pods --show-labels
kubectl patch service web -p '{"spec":{"selector":{"app":"web"}}}'
kubectl get endpoints web
```

What to learn:

```text
Services route to Pods through selectors.
No endpoints usually means label mismatch or unready Pods.
```

### Scenario 3: Create an Ingress Rule

Task:

```text
Expose a Service through an Ingress and test it with a Host header.
```

Commands:

```bash
kubectl create deployment web --image=nginx
kubectl expose deployment web --port=80
kubectl apply -f ingress.yaml
kubectl get ingress
curl -H "Host: web.example.com" http://<ingress-ip>
```

What to learn:

```text
Ingress routes HTTP by host and path.
The ingress controller is the real traffic handler.
```

### Scenario 4: Debug Ingress 503

Task:

```text
Break the Service selector behind an Ingress. Use Ingress, Service, EndpointSlice, and Pod checks to find the cause.
```

Commands:

```bash
kubectl describe ingress web
kubectl get svc web -o yaml
kubectl get endpoints web
kubectl get endpointslice -l kubernetes.io/service-name=web
kubectl get pods --show-labels
```

What to learn:

```text
Ingress can be correct while the backend Service is broken.
Always check the path from Ingress to Service to endpoints to Pods.
```

### Scenario 5: TLS Secret Practice

Task:

```text
Create a TLS Secret and reference it from an Ingress.
```

Commands:

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=web.example.com"
kubectl create secret tls web-tls --cert=tls.crt --key=tls.key
kubectl apply -f ingress-tls.yaml
kubectl describe ingress web
curl -vk https://web.example.com
```

What to learn:

```text
Ingress TLS requires a Secret with certificate and key.
The host in the Ingress should match the certificate name.
```

## Exam and Interview Tips

| Question | Good answer |
| --- | --- |
| What provides Kubernetes DNS? | CoreDNS. |
| What DNS name should work across namespaces? | `<service>.<namespace>.svc.cluster.local`. |
| What object tells a Service which Pods to route to? | EndpointSlice, created from matching Service selectors and Pod readiness. |
| What usually causes a Service to have no endpoints? | Selector mismatch or unready Pods. |
| Does Ingress work without a controller? | No. Ingress is only the rule object. |
| What selects the Ingress controller? | `ingressClassName` or controller-specific default behavior. |
| What does Ingress route on? | HTTP host and path. |
| What stores Ingress TLS certs? | A Kubernetes TLS Secret. |
| How do you test local Ingress without DNS? | Curl the ingress IP with a `Host` header. |
