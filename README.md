# Quick References

A collection of short command-line quick references and study guides for common developer, DevOps, platform engineering, and Kubernetes certification topics.

## Quick References

| Guide | Covers |
| --- | --- |
| [AWS CLI, CDK, and kubectl](aws_cdk_kubectl_quick_reference.md) | AWS profiles, S3, EC2, IAM, CloudFormation, ECR, CDK, basic kubectl, Kustomize, and ArgoCD commands. |
| [Cloud Networking](cloud_networking_quick_reference.md) | DNS, TLS, HTTP checks, Linux network debugging, AWS VPC, load balancing, Route 53, and Kubernetes networking. |
| [Database Ops](database_ops_quick_reference.md) | PostgreSQL, MySQL, backups, restores, slow queries, locks, Kubernetes database access, AWS RDS, and common triage. |
| [Docker and Docker Compose](docker_dockercompose_quick_reference.md) | Images, containers, logs, exec, networks, volumes, Compose lifecycle, rebuilds, validation, and cleanup commands. |
| [GitHub Actions and GitLab CI](github_gitlab_ci_quick_reference.md) | GitHub Actions CLI, workflow syntax, permissions, matrix jobs, caches, artifacts, GitLab CI rules, and CI debugging. |
| [Helm](helm_quick_reference.md) | Repositories, installs, upgrades, releases, charts, templates, values, debugging, and OCI registries. |
| [kubectl Imperative](kubectl_imperative_quick_reference.md) | Dry-run YAML generation, pods, deployments, services, ConfigMaps, Secrets, Jobs, labels, annotations, namespaces, and fast edit flows. |
| [Kubernetes](kubernetes_quick_reference.md) | kubectl contexts, namespaces, workloads, logs, rollouts, networking, config, storage, RBAC, Helm, kubeadm, and troubleshooting. |
| [Kubernetes Manifests](kubernetes_manifests_quick_reference.md) | YAML examples for Pods, Deployments, Services, ConfigMaps, Secrets, probes, resources, volumes, PVCs, Jobs, CronJobs, and Ingress. |
| [Kubernetes Troubleshooting](kubernetes_troubleshooting_quick_reference.md) | First-look commands, pod states, rollouts, services, DNS, nodes, cluster components, kubeadm host checks, storage, and NetworkPolicy. |
| [Linux Command Line](linux_commands_quick_reference.md) | System info, files, search, disk and memory, networking, permissions, processes, packages, archives, SSH, systemd, and shortcuts. |
| [Observability](observability_quick_reference.md) | Prometheus, PromQL, Grafana, Loki, OpenTelemetry, SLO checks, alerts, and Kubernetes troubleshooting views. |
| [Terraform and OpenTofu](terraform_opentofu_quick_reference.md) | Init, format, validate, plan, apply, variables, state, import, drift, workspaces, modules, targeting, and debugging. |
| [Vim Commands](vim_commands_quick_reference.md) | Modes, save and quit, movement, editing, search and replace, visual selections, buffers, windows, tabs, marks, jumps, and macros. |

## Study Guides

| Guide | Covers |
| --- | --- |
| [CKA and CKAD Exam Speed Sheet](cka_ckad_exam_speed_sheet.md) | Exam aliases, Vim setup, context switching, YAML generation, `kubectl explain`, rollouts, services, RBAC, cleanup, and time management. |
| [Kubernetes Definitions Study Sheet](kubernetes_definitions_study_sheet.md) | Diagram-backed study tables for control plane, node components, workloads, networking, storage, security, scheduling, probes, Helm, Kustomize, CRDs, and operators. |
| [Minikube CKA Practice](minikube_cka_quick_reference.md) | Local two-node Minikube practice setup for CKA/CKAD labs, addons, node access, service access, and rebuild commands. |
| [strace Study Sheet](strace_study_sheet.md) | System call tracing concepts, common options, file and network errors, hung process triage, Kubernetes/container usage, output reading, and safety notes. |

## Suggested Starting Points

```text
New Linux shell usage       -> linux_commands_quick_reference.md
Docker local development    -> docker_dockercompose_quick_reference.md
AWS and CDK work            -> aws_cdk_kubectl_quick_reference.md
Infrastructure as code      -> terraform_opentofu_quick_reference.md
CI/CD pipelines             -> github_gitlab_ci_quick_reference.md
Kubernetes administration   -> kubernetes_quick_reference.md
Kubernetes definitions      -> kubernetes_definitions_study_sheet.md
Exam speed and aliases      -> cka_ckad_exam_speed_sheet.md
Fast kubectl generation     -> kubectl_imperative_quick_reference.md
Kubernetes YAML examples    -> kubernetes_manifests_quick_reference.md
Kubernetes troubleshooting  -> kubernetes_troubleshooting_quick_reference.md
Kubernetes packaging        -> helm_quick_reference.md
CKA local practice          -> minikube_cka_quick_reference.md
Observability and alerts    -> observability_quick_reference.md
Cloud connectivity          -> cloud_networking_quick_reference.md
Database operations         -> database_ops_quick_reference.md
System call debugging       -> strace_study_sheet.md
Terminal editing            -> vim_commands_quick_reference.md
```

## Naming

Each guide is a standalone Markdown file named after the topic it covers. Open the file directly when you need commands for that tool.
