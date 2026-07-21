# Helm Quick Reference Guide

## Repositories
```bash
helm repo list
helm repo add <name> <url>
helm repo update
helm search repo <chart>
helm search repo <chart> --versions
helm repo remove <name>
```

## Install and Upgrade
```bash
helm install <release> <chart>
helm install <release> <chart> -n <namespace> --create-namespace
helm upgrade <release> <chart>
helm upgrade --install <release> <chart>
helm upgrade --install <release> <chart> -f values.yaml
helm upgrade --install <release> <chart> --set image.tag=<tag>
```

## Releases
```bash
helm list
helm list -A
helm status <release>
helm status <release> -n <namespace>
helm history <release>
helm rollback <release> <revision>
helm uninstall <release>
helm uninstall <release> -n <namespace>
```

## Charts
```bash
helm create <chart>
helm dependency list
helm dependency update
helm lint <chart>
helm package <chart>
helm show chart <chart>
helm show values <chart>
helm show all <chart>
```

## Templates and Debugging
```bash
helm template <release> <chart>
helm template <release> <chart> -f values.yaml
helm install <release> <chart> --dry-run --debug
helm upgrade <release> <chart> --dry-run --debug
helm get values <release>
helm get manifest <release>
helm get all <release>
```

## Values
```bash
helm upgrade --install <release> <chart> -f values.yaml
helm upgrade --install <release> <chart> -f base.yaml -f prod.yaml
helm upgrade --install <release> <chart> --set key=value
helm upgrade --install <release> <chart> --set-string key=value
helm upgrade --install <release> <chart> --set-file config=config.yaml
```

## OCI Registries
```bash
helm registry login <registry>
helm pull oci://<registry>/<chart> --version <version>
helm push <chart>.tgz oci://<registry>/<repo>
helm install <release> oci://<registry>/<chart> --version <version>
```
