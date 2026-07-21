# Observability Quick Reference Guide

## Prometheus
```bash
kubectl get pods -A | grep prometheus
kubectl port-forward svc/prometheus-server 9090:80 -n monitoring
promtool check rules rules.yaml
promtool check config prometheus.yml
```

## PromQL
```promql
up
rate(http_requests_total[5m])
sum(rate(container_cpu_usage_seconds_total[5m])) by (pod)
sum(container_memory_working_set_bytes) by (pod)
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))
kube_pod_container_status_restarts_total
```

## Grafana
```bash
kubectl get svc -A | grep grafana
kubectl port-forward svc/grafana 3000:80 -n monitoring
```

## Loki and Logs
```bash
kubectl logs <pod>
kubectl logs <pod> -c <container>
kubectl logs -f <pod>
kubectl logs --previous <pod>
kubectl logs -l app=<label> --tail=100
logcli labels
logcli query '{namespace="default"}'
```

## OpenTelemetry
```bash
otelcol --version
kubectl get pods -A | grep otel
kubectl logs deployment/otel-collector -n observability
kubectl port-forward svc/otel-collector 4317:4317 -n observability
kubectl port-forward svc/otel-collector 4318:4318 -n observability
```

## SLO and Alert Checks
```promql
sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))
increase(kube_pod_container_status_restarts_total[1h])
avg_over_time(up[5m])
```

## Kubernetes Troubleshooting Views
```bash
kubectl top nodes
kubectl top pods -A
kubectl get events -A --sort-by=.metadata.creationTimestamp
kubectl describe pod <pod>
kubectl get --raw /metrics
kubectl get --raw /readyz
kubectl get --raw /livez
```
