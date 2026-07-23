# Kubernetes Ingress Gateway Scenarios

Use this guide to practice real ingress gateway problems. The goal is to understand the path from an external request to a Pod, know which object owns each part of the flow, and debug failures quickly.

This guide complements [Kubernetes DNS and Ingress Study Guide](kubernetes_dns_ingress_study_guide.md). That guide covers the basics; this one focuses on scenarios.

## Mental Model

```text
Client
  -> DNS record
  -> external load balancer / NodePort
  -> ingress gateway Service
  -> ingress gateway Pod
  -> Gateway / Ingress routing config
  -> backend Service
  -> EndpointSlice
  -> ready Pod
```

Gateway API model:

```text
GatewayClass -> Gateway listener -> HTTPRoute -> Service -> EndpointSlice -> Pod
```

Istio API model:

```text
istio-ingressgateway Service/Pod -> Gateway -> VirtualService -> Service -> Pod
```

## Key Terms

| Term | Meaning | Study note |
| --- | --- | --- |
| Ingress | Kubernetes object for HTTP/HTTPS routing to Services. | Requires an Ingress controller. |
| Ingress controller | Proxy/controller that implements Ingress objects. | Examples: NGINX, Traefik, HAProxy, cloud controllers. |
| Gateway API | Newer Kubernetes networking API for L4/L7 routing. | Uses `GatewayClass`, `Gateway`, and protocol-specific Routes. |
| GatewayClass | Cluster-scoped object describing the gateway controller type. | Similar idea to `StorageClass`, but for gateways. |
| Gateway | Defines listener ports, protocols, hostnames, TLS, and attachment policy. | Usually owned by platform/SRE teams. |
| HTTPRoute | Defines HTTP host/path/header matching and backend routing. | Usually owned by app teams. |
| Istio Gateway | Istio resource that configures ingress gateway listeners. | Routing lives in `VirtualService`. |
| VirtualService | Istio routing object for hosts, paths, weights, rewrites, retries, and more. | Pairs with Istio `Gateway` for ingress. |
| Listener | A port/protocol/hostname entry on a Gateway. | Example: HTTP port 80 or HTTPS port 443. |
| BackendRef | Gateway API reference to a backend, usually a Service. | Can include weights for traffic splitting. |
| ReferenceGrant | Gateway API object that permits cross-namespace references. | Needed when a Route points to a backend in another namespace. |
| TLS termination | Gateway decrypts HTTPS and sends HTTP or HTTPS to backend. | Needs a TLS Secret. |
| TLS passthrough | Gateway passes encrypted traffic through without decrypting. | Often SNI-based and implementation-specific. |

## Gateway API vs Ingress vs Istio Gateway

| Need | Common fit | Why |
| --- | --- | --- |
| Basic host/path routing. | Ingress | Widely supported and simple. |
| Standard Kubernetes API with richer routing. | Gateway API | More expressive and role-oriented than Ingress. |
| Service mesh ingress with mesh traffic policy. | Istio Gateway + VirtualService or Gateway API with Istio | Integrates with Istio routing, telemetry, mTLS, and policy. |
| Weighted canary at ingress. | Gateway API or Istio | Native weighted routing support. |
| Header-based routing. | Gateway API or Istio | Avoids controller-specific Ingress annotations. |
| App teams attach routes to shared edge gateways. | Gateway API | Gateway and Route resources split responsibilities. |

## Baseline Checks

Run these before deep debugging:

```bash
kubectl get ingress -A
kubectl get ingressclass
kubectl get gatewayclass
kubectl get gateway -A
kubectl get httproute -A
kubectl get svc -A | grep -Ei 'ingress|gateway|proxy|loadbalancer'
kubectl get pods -A | grep -Ei 'ingress|gateway|proxy'
```

Check the outside address:

```bash
kubectl get svc -A -o wide | grep -Ei 'LoadBalancer|NodePort|ingress|gateway'
kubectl get ingress -A -o wide
kubectl get gateway -A -o wide
```

Check routes and status:

```bash
kubectl describe ingress <name> -n <namespace>
kubectl describe gateway <name> -n <namespace>
kubectl describe httproute <name> -n <namespace>
```

## Scenario 1: Basic Gateway API HTTP Route

Goal: expose a `web` Service through a Gateway API `Gateway` and `HTTPRoute`.

App:

```bash
kubectl create namespace gateway-lab
kubectl create deployment web --image=nginx -n gateway-lab
kubectl expose deployment web --port=80 -n gateway-lab
```

Gateway:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: shared-gateway
  namespace: gateway-lab
spec:
  gatewayClassName: <gateway-class>
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    hostname: web.example.com
```

HTTPRoute:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: web
  namespace: gateway-lab
spec:
  parentRefs:
  - name: shared-gateway
  hostnames:
  - web.example.com
  rules:
  - backendRefs:
    - name: web
      port: 80
```

Verify:

```bash
kubectl get gateway shared-gateway -n gateway-lab
kubectl get httproute web -n gateway-lab
kubectl describe httproute web -n gateway-lab
kubectl get endpointslice -n gateway-lab -l kubernetes.io/service-name=web
curl -H 'Host: web.example.com' http://<gateway-address>/
```

Expected result: the `HTTPRoute` is accepted and traffic reaches nginx.

## Scenario 2: Host Routing to Two Apps

Goal: route different hostnames through the same gateway.

```text
web.example.com -> web Service
api.example.com -> api Service
```

Create apps:

```bash
kubectl create deployment web --image=nginx -n gateway-lab
kubectl expose deployment web --port=80 -n gateway-lab
kubectl create deployment api --image=httpd -n gateway-lab
kubectl expose deployment api --port=80 -n gateway-lab
```

Use two `HTTPRoute` objects:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: web
  namespace: gateway-lab
spec:
  parentRefs:
  - name: shared-gateway
  hostnames:
  - web.example.com
  rules:
  - backendRefs:
    - name: web
      port: 80
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: api
  namespace: gateway-lab
spec:
  parentRefs:
  - name: shared-gateway
  hostnames:
  - api.example.com
  rules:
  - backendRefs:
    - name: api
      port: 80
```

Test:

```bash
curl -H 'Host: web.example.com' http://<gateway-address>/
curl -H 'Host: api.example.com' http://<gateway-address>/
```

Debug if both hosts hit the same app:

```bash
kubectl describe gateway shared-gateway -n gateway-lab
kubectl describe httproute web -n gateway-lab
kubectl describe httproute api -n gateway-lab
```

## Scenario 3: Path Routing

Goal: route `/api` to one Service and `/` to another Service.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: path-routing
  namespace: gateway-lab
spec:
  parentRefs:
  - name: shared-gateway
  hostnames:
  - app.example.com
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api
    backendRefs:
    - name: api
      port: 80
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: web
      port: 80
```

Test:

```bash
curl -H 'Host: app.example.com' http://<gateway-address>/
curl -H 'Host: app.example.com' http://<gateway-address>/api
```

Study note: path matching behavior can vary at the edges between controllers. In exams and interviews, explain the rule order and verify with `curl`.

## Scenario 4: Weighted Canary Routing

Goal: send most traffic to `web-v1` and a small amount to `web-v2`.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: web-canary
  namespace: gateway-lab
spec:
  parentRefs:
  - name: shared-gateway
  hostnames:
  - web.example.com
  rules:
  - backendRefs:
    - name: web-v1
      port: 80
      weight: 90
    - name: web-v2
      port: 80
      weight: 10
```

Verify:

```bash
for i in $(seq 1 20); do
  curl -s -H 'Host: web.example.com' http://<gateway-address>/ | head -1
done
```

Checks:

```bash
kubectl get svc web-v1 web-v2 -n gateway-lab
kubectl get endpointslice -n gateway-lab -l kubernetes.io/service-name=web-v1
kubectl get endpointslice -n gateway-lab -l kubernetes.io/service-name=web-v2
kubectl describe httproute web-canary -n gateway-lab
```

## Scenario 5: TLS Termination

Goal: terminate HTTPS at the gateway.

Create or provide a TLS Secret:

```bash
kubectl create secret tls web-tls \
  --cert=tls.crt \
  --key=tls.key \
  -n gateway-lab
```

Gateway listener:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: shared-gateway
  namespace: gateway-lab
spec:
  gatewayClassName: <gateway-class>
  listeners:
  - name: https
    protocol: HTTPS
    port: 443
    hostname: web.example.com
    tls:
      mode: Terminate
      certificateRefs:
      - name: web-tls
```

Test:

```bash
curl -vk --resolve web.example.com:443:<gateway-ip> https://web.example.com/
```

Common failures:

| Symptom | Likely cause | Check |
| --- | --- | --- |
| TLS handshake fails. | Bad Secret, unsupported listener config, wrong cert key pair. | `kubectl describe gateway`. |
| Browser shows wrong certificate. | Hostname or listener conflict. | Gateway listeners and certificate SANs. |
| HTTP works but HTTPS fails. | Port 443 listener or load balancer missing. | Gateway Service ports. |

## Scenario 6: Cross-Namespace Route Attachment

Goal: app teams define routes in their namespace while platform owns the shared gateway.

Platform namespace:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: shared-gateway
  namespace: platform-ingress
spec:
  gatewayClassName: <gateway-class>
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      namespaces:
        from: Selector
        selector:
          matchLabels:
            shared-gateway-access: "true"
```

App namespace:

```bash
kubectl label namespace app-a shared-gateway-access=true
```

App route:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: app-a
  namespace: app-a
spec:
  parentRefs:
  - name: shared-gateway
    namespace: platform-ingress
  hostnames:
  - app-a.example.com
  rules:
  - backendRefs:
    - name: app-a
      port: 80
```

Debug attachment:

```bash
kubectl describe gateway shared-gateway -n platform-ingress
kubectl describe httproute app-a -n app-a
kubectl get namespace app-a --show-labels
```

Expected result: the route status shows it is accepted by the parent Gateway.

## Scenario 7: Cross-Namespace Backend

Goal: route in one namespace points to a Service in another namespace.

HTTPRoute:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: frontend
  namespace: frontend
spec:
  parentRefs:
  - name: shared-gateway
    namespace: platform-ingress
  rules:
  - backendRefs:
    - name: api
      namespace: backend
      port: 80
```

The backend namespace must allow the reference:

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-frontend-to-api
  namespace: backend
spec:
  from:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    namespace: frontend
  to:
  - group: ""
    kind: Service
    name: api
```

Debug:

```bash
kubectl describe httproute frontend -n frontend
kubectl get referencegrant -n backend
kubectl get svc api -n backend
kubectl get endpointslice -n backend -l kubernetes.io/service-name=api
```

Expected failure without `ReferenceGrant`: route reference resolution fails or backend is not accepted.

## Scenario 8: Istio Ingress Gateway with Gateway and VirtualService

Goal: expose a Service through Istio ingress.

Gateway:

```yaml
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  name: web-gateway
  namespace: gateway-lab
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - web.example.com
```

VirtualService:

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: web
  namespace: gateway-lab
spec:
  hosts:
  - web.example.com
  gateways:
  - web-gateway
  http:
  - route:
    - destination:
        host: web.gateway-lab.svc.cluster.local
        port:
          number: 80
```

Find the ingress address:

```bash
kubectl get svc -n istio-system istio-ingressgateway
kubectl get pods -n istio-system -l istio=ingressgateway
```

Test:

```bash
curl -H 'Host: web.example.com' http://<ingress-address>/
```

Debug:

```bash
kubectl describe gateway web-gateway -n gateway-lab
kubectl describe virtualservice web -n gateway-lab
kubectl logs -n istio-system -l istio=ingressgateway --tail=100
kubectl get svc web -n gateway-lab
kubectl get endpointslice -n gateway-lab -l kubernetes.io/service-name=web
```

## Scenario 9: Gateway Returns 404

Likely meaning: the request reached the gateway, but no route matched.

Check:

```bash
curl -v -H 'Host: web.example.com' http://<gateway-address>/missing
kubectl get httproute -A
kubectl describe httproute <route> -n <namespace>
kubectl describe gateway <gateway> -n <namespace>
```

Common causes:

| Cause | Fix |
| --- | --- |
| Wrong `Host` header. | Use correct hostname or update route hostnames. |
| Path does not match. | Check path type and path value. |
| Route is not attached to Gateway. | Check `parentRefs` and Gateway `allowedRoutes`. |
| Istio `VirtualService.gateways` missing the ingress Gateway. | Add the Gateway name to `gateways`. |
| Gateway listener hostname does not allow route hostname. | Align listener and route hostnames. |

## Scenario 10: Gateway Returns 503

Likely meaning: route matched, but no healthy backend was available.

Check backend path:

```bash
kubectl get svc <service> -n <namespace> -o yaml
kubectl get endpointslice -n <namespace> -l kubernetes.io/service-name=<service>
kubectl get pods -n <namespace> --show-labels
kubectl describe pod <pod> -n <namespace>
kubectl describe httproute <route> -n <namespace>
```

Common causes:

| Cause | Fix |
| --- | --- |
| Service selector does not match Pods. | Patch Service selector or Pod labels. |
| Pods are not Ready. | Fix readiness probe or app startup. |
| Route points to wrong Service port. | Use the Service `port`, not the container port unless they match. |
| Backend reference is not resolved. | Check route status and ReferenceGrant. |
| Istio destination host is wrong. | Use full Service DNS name if namespace is ambiguous. |

## Scenario 11: Gateway Address Is Empty

Likely meaning: the controller has not provisioned or reported an address.

Check:

```bash
kubectl get gateway -A
kubectl describe gateway <gateway> -n <namespace>
kubectl get gatewayclass
kubectl describe gatewayclass <gateway-class>
kubectl get pods -A | grep -Ei 'gateway|ingress|controller'
kubectl get events -A --sort-by=.lastTimestamp | tail -50
```

Common causes:

| Cause | Fix |
| --- | --- |
| Wrong `gatewayClassName`. | Use an installed GatewayClass. |
| Gateway controller is not installed. | Install or repair the controller. |
| LoadBalancer integration is unavailable. | Use NodePort, port-forward, MetalLB, minikube tunnel, or cloud LB setup. |
| Controller lacks permissions. | Check controller logs and RBAC. |

## Scenario 12: Local Minikube Gateway Practice

Options depend on the gateway controller you install.

For standard Ingress practice:

```bash
minikube addons enable ingress
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
minikube tunnel
```

For Gateway API practice:

```bash
kubectl get crd gateways.gateway.networking.k8s.io
kubectl get gatewayclass
kubectl get gateway -A
kubectl get httproute -A
```

If your local cluster has no GatewayClass, install a Gateway API implementation such as Istio, Envoy Gateway, Contour, or another supported controller.

## Fast Troubleshooting Flow

Use this order:

```text
1. DNS: does the hostname resolve to the expected external address?
2. Gateway exposure: does the LoadBalancer, NodePort, or port-forward work?
3. Gateway object: is the listener programmed and accepted?
4. Route object: is the HTTPRoute, Ingress, or VirtualService attached?
5. Host/path match: does the request match the configured host and path?
6. Backend Service: does the route point to the right Service and port?
7. EndpointSlice: does the Service have ready endpoints?
8. Pods: are backend Pods Ready and listening on the expected port?
9. Policy: is NetworkPolicy, mesh auth, TLS, or mTLS blocking traffic?
10. Logs: what does the gateway controller or proxy log say?
```

Commands:

```bash
dig +short <host>
curl -vk -H 'Host: <host>' http://<gateway-address>/<path>
kubectl describe gateway <gateway> -n <namespace>
kubectl describe httproute <route> -n <namespace>
kubectl describe ingress <ingress> -n <namespace>
kubectl get svc <service> -n <namespace> -o yaml
kubectl get endpointslice -n <namespace> -l kubernetes.io/service-name=<service>
kubectl get pods -n <namespace> -o wide --show-labels
kubectl logs -n <gateway-namespace> -l <gateway-label-selector> --tail=100
```

## Common Interview Questions

| Question | Strong answer |
| --- | --- |
| What is the difference between Ingress and an Ingress controller? | Ingress is the API object with routing rules; the controller is the running proxy/controller that implements those rules. |
| What is Gateway API trying to improve? | It separates infrastructure and app routing roles, supports richer routing, and avoids many controller-specific Ingress annotations. |
| What is the difference between Gateway and HTTPRoute? | Gateway owns listeners and attachment policy; HTTPRoute owns HTTP routing rules to Services. |
| Why would a route show accepted false? | Parent Gateway does not allow it, hostnames conflict, references fail, or the controller rejects unsupported config. |
| Why does an ingress gateway return 404? | The gateway was reached, but no host/path route matched. |
| Why does an ingress gateway return 503? | The route matched, but the backend Service has no healthy endpoints or the backend reference is invalid. |
| Why use ReferenceGrant? | To let a backend namespace explicitly allow routes from another namespace to reference its Services. |
| Where do labels matter in ingress troubleshooting? | Services select Pods by labels, Gateway allowedRoutes can select namespaces by labels, and gateway Pods are often selected by labels. |

## Official References

| Topic | Reference |
| --- | --- |
| Gateway API introduction | https://gateway-api.sigs.k8s.io/docs/introduction/ |
| Gateway API HTTPRoute | https://gateway-api.sigs.k8s.io/reference/api-types/httproute/ |
| Gateway API specification | https://gateway-api.sigs.k8s.io/reference/api-spec/main/spec/ |
| Istio ingress gateways | https://istio.io/latest/docs/tasks/traffic-management/ingress/ingress-control/ |
| Istio with Kubernetes Gateway API | https://istio.io/latest/docs/tasks/traffic-management/ingress/gateway-api/ |
