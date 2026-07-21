# Kubernetes Definitions Study Sheet

## Control Plane

### kube-apiserver
The front door to the Kubernetes control plane. It exposes the Kubernetes API, validates requests, handles authentication and authorization, and stores accepted changes through etcd.

### etcd
The strongly consistent key-value database that stores Kubernetes cluster state, including objects such as Pods, Deployments, Secrets, ConfigMaps, and Nodes.

### kube-scheduler
The control-plane component that assigns newly created Pods to Nodes. It considers resource requests, node selectors, affinity, taints, tolerations, topology, and other scheduling constraints.

### kube-controller-manager
The control-plane component that runs controller loops. A controller watches the current cluster state, compares it to the desired state, and makes changes to move the cluster toward the desired state.

Examples of controllers include:

```text
Node controller
Deployment controller
ReplicaSet controller
Job controller
EndpointSlice controller
ServiceAccount controller
Namespace controller
```

### cloud-controller-manager
The control-plane component that connects Kubernetes to a cloud provider. It handles cloud-specific behavior such as load balancers, node metadata, and cloud routes when the cluster uses an external cloud integration.

## Node Components

### Node
A worker machine in the cluster. A Node can be a virtual machine or physical machine and runs Pods.

### kubelet
The agent that runs on each Node. It watches PodSpecs assigned to the Node and makes sure the required containers are running through the container runtime.

### kube-proxy
The node-level networking component that implements Service virtual IP behavior and load balancing to backend Pods, usually using iptables, IPVS, or nftables rules.

### container runtime
The software that runs containers on a Node. Common examples include containerd and CRI-O.

### CRI
Container Runtime Interface. The Kubernetes API between kubelet and the container runtime.

## Workloads

### Pod
The smallest deployable unit in Kubernetes. A Pod wraps one or more containers that share networking and storage.

### ReplicaSet
Ensures that a specified number of matching Pods are running. Deployments create and manage ReplicaSets during rollouts.

### Deployment
Manages stateless replicated Pods and supports rolling updates, rollbacks, scaling, and rollout history.

### StatefulSet
Manages stateful applications that need stable network identities, stable storage, and ordered rollout or termination.

### DaemonSet
Ensures that a copy of a Pod runs on every matching Node, or on selected Nodes. Common uses include log agents, monitoring agents, and CNI components.

### Job
Runs Pods until a task completes successfully.

### CronJob
Runs Jobs on a schedule.

## Networking

### Service
Provides a stable virtual IP and DNS name for accessing a set of Pods selected by labels.

### ClusterIP
The default Service type. It exposes a Service only inside the cluster.

### NodePort
Exposes a Service on a port across each Node, allowing traffic to reach the Service through `<node-ip>:<node-port>`.

### LoadBalancer
Exposes a Service through an external load balancer, usually created by the cloud provider.

### EndpointSlice
Tracks the network endpoints backing a Service. EndpointSlices point to the Pods or addresses that should receive traffic.

### Ingress
Defines HTTP and HTTPS routing rules from outside the cluster to Services inside the cluster. It requires an Ingress controller to do the actual routing.

### Ingress controller
The controller that watches Ingress objects and configures a proxy or load balancer to route traffic.

### NetworkPolicy
Defines allowed network traffic between Pods, namespaces, and IP blocks. It requires a CNI plugin that enforces NetworkPolicy.

### CNI
Container Network Interface. The plugin system Kubernetes uses for Pod networking.

### CoreDNS
The DNS server used by Kubernetes for cluster DNS. It resolves Service names and Pod DNS records.

## Configuration and Secrets

### ConfigMap
Stores non-sensitive configuration data as key-value pairs or files.

### Secret
Stores sensitive configuration data such as passwords, tokens, and keys. Secrets are base64-encoded in manifests but are not encrypted by default unless encryption at rest is configured.

### ServiceAccount
An identity for processes running in Pods. It is commonly used with RBAC to grant Pods access to the Kubernetes API.

## Storage

### Volume
Storage attached to a Pod. A volume can be temporary, config-backed, secret-backed, host-backed, or persistent.

### PersistentVolume
A cluster-level storage resource.

### PersistentVolumeClaim
A request for storage by a workload. A PVC binds to a matching PersistentVolume or triggers dynamic provisioning.

### StorageClass
Defines how dynamic storage should be provisioned.

### access mode
Defines how a volume can be mounted, such as `ReadWriteOnce`, `ReadOnlyMany`, or `ReadWriteMany`.

### reclaim policy
Defines what happens to a PersistentVolume after its claim is deleted. Common policies are `Retain` and `Delete`.

## Security

### RBAC
Role-Based Access Control. It controls who can do what against Kubernetes API resources.

### Role
Grants permissions within a namespace.

### ClusterRole
Grants permissions cluster-wide, or defines reusable permissions that can be bound inside a namespace.

### RoleBinding
Binds a Role or ClusterRole to users, groups, or ServiceAccounts inside a namespace.

### ClusterRoleBinding
Binds a ClusterRole to users, groups, or ServiceAccounts across the cluster.

### SecurityContext
Defines security settings for a Pod or container, such as user ID, group ID, privilege mode, Linux capabilities, and read-only root filesystem.

### admission controller
A control-plane plugin that can validate or mutate API requests after authentication and authorization but before persistence.

## Scheduling

### resource request
The amount of CPU or memory Kubernetes reserves for a container when scheduling.

### resource limit
The maximum CPU or memory a container is allowed to use.

### taint
A Node property that repels Pods unless they tolerate it.

### toleration
A Pod property that allows the Pod to be scheduled onto a Node with a matching taint.

### nodeSelector
A simple Pod scheduling rule that requires a Node to have matching labels.

### affinity
Advanced scheduling rules that attract Pods to Nodes or to other Pods.

### anti-affinity
Scheduling rules that keep Pods away from certain Nodes or away from other Pods.

## Health and Lifecycle

### livenessProbe
Checks whether a container is still alive. If it fails repeatedly, kubelet restarts the container.

### readinessProbe
Checks whether a container is ready to receive traffic. If it fails, the Pod is removed from Service endpoints.

### startupProbe
Checks whether a slow-starting container has finished starting. While it is failing, liveness and readiness checks are delayed.

### finalizer
A field that prevents Kubernetes from deleting an object until cleanup logic has completed.

### ownerReference
Metadata that links an object to its owning object. It helps Kubernetes garbage-collect dependent resources.

## Packaging and Customization

### Helm
A Kubernetes package manager that installs and manages applications using charts.

### Chart
A Helm package containing templates, default values, metadata, and optional dependencies.

### values.yaml
The Helm values file used to configure a chart before rendering manifests.

### Kustomize
A Kubernetes-native customization tool that builds manifests from bases, overlays, patches, generators, and transformers.

### CRD
CustomResourceDefinition. It extends the Kubernetes API with a new resource type.

### custom resource
An object created from a CRD.

### operator
A controller that manages an application or platform using Kubernetes custom resources and reconciliation loops.
