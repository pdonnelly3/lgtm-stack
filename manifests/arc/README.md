# Monitoring GitHub Actions Self-Hosted Runners (ARC) with LGTM Stack

This guide explains how to monitor GitHub Actions self-hosted runners deployed with the Actions Runner Controller (ARC) using the LGTM stack.

## Prerequisites

- LGTM stack installed and running (see main [README.md](../../README.md))
- ARC operator installed in your Kubernetes cluster
- Promtail deployed and collecting container logs

## Architecture Overview

```
GitHub Actions Runners (ARC)
    ↓
    ├─→ Logs → Promtail → Loki
    ├─→ Metrics → Prometheus → Mimir
    └─→ Resource Usage → kube-state-metrics → Prometheus → Mimir
```

## Setup Steps

### 1. Configure ARC Controller for Monitoring

When installing the ARC controller, add these values to expose metrics:

```bash
# Create namespace for ARC controller
kubectl create namespace arc-systems

# Create GitHub token secret
kubectl create secret generic github-token \
  --namespace=arc-runners \
  --from-literal=github_token='YOUR_GITHUB_PAT'

# Install ARC controller with monitoring enabled
helm install arc \
  --namespace arc-systems \
  --create-namespace \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller \
  --set metrics.controllerManagerAddr=":8080" \
  --set metrics.listenerAddr=":8080" \
  --set metrics.listenerEndpoint="/metrics"
```

### 2. Configure Runner Scale Set

Install your runner scale set with proper labels and logging configuration:

```bash
# Create namespace for runners
kubectl create namespace arc-runners

# Install runner scale set
helm install arc-runner-set \
  --namespace arc-runners \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set \
  --set githubConfigUrl="https://github.com/YOUR_ORG/YOUR_REPO" \
  --set githubConfigSecret.github_token="YOUR_GITHUB_PAT" \
  --set template.spec.containers[0].env[0].name="ACTIONS_RUNNER_PRINT_LOG_TO_STDOUT" \
  --set template.spec.containers[0].env[0].value="1"
```

**Or use a values file** (see [values-arc-monitoring.yaml](values-arc-monitoring.yaml) for a complete example).

### 3. Deploy PodMonitor for Metrics Collection

Apply the PodMonitor to enable Prometheus scraping:

```bash
kubectl apply -f manifests/arc/podmonitor-arc.yaml
```

This creates:
- **PodMonitor** for runner pods
- **ServiceMonitor** for ARC controller

### 4. Verify Setup

Check that everything is configured correctly:

```bash
# Check runner pods are running
kubectl get pods -n arc-runners

# Check PodMonitor is created
kubectl get podmonitor -n arc-runners

# Check ServiceMonitor is created
kubectl get servicemonitor -n arc-systems

# Verify Prometheus is scraping targets
kubectl port-forward -n monitoring svc/prometheus-operator-kube-prom-prometheus 9090:9090
# Open http://localhost:9090/targets and look for arc-runners targets
```

## Monitoring in Grafana

### Access Grafana

```bash
kubectl port-forward svc/lgtm-grafana 3000:80 -n monitoring
# Username: admin
# Password: make get-grafana-password
```

### Logs (Loki)

Query runner logs in Grafana Explore:

```logql
# All runner logs
{namespace="arc-runners"}

# Filter by pod
{namespace="arc-runners", pod=~"arc-runner-.*"}

# Search for specific job patterns
{namespace="arc-runners"} |= "job"

# Filter by log level (if structured logging is enabled)
{namespace="arc-runners"} | json | level="error"

# Show job execution times
{namespace="arc-runners"} |= "Job" | pattern `<_> Job <job_id> <status>`
```

### Metrics (Mimir)

Query runner metrics in Grafana Explore:

```promql
# CPU usage by runner pod
rate(container_cpu_usage_seconds_total{namespace="arc-runners"}[5m])

# Memory usage by runner pod
container_memory_usage_bytes{namespace="arc-runners"}

# Pod count
count(kube_pod_info{namespace="arc-runners"})

# Runner pod restarts
rate(kube_pod_container_status_restarts_total{namespace="arc-runners"}[5m])

# Pods in non-running state
count(kube_pod_status_phase{namespace="arc-runners", phase!="Running"}) by (phase)

# Container resource requests vs limits
sum(kube_pod_container_resource_requests{namespace="arc-runners", resource="cpu"}) by (pod)
sum(kube_pod_container_resource_limits{namespace="arc-runners", resource="cpu"}) by (pod)

# Memory requests vs limits
sum(kube_pod_container_resource_requests{namespace="arc-runners", resource="memory"}) by (pod)
sum(kube_pod_container_resource_limits{namespace="arc-runners", resource="memory"}) by (pod)

# Network I/O
rate(container_network_receive_bytes_total{namespace="arc-runners"}[5m])
rate(container_network_transmit_bytes_total{namespace="arc-runners"}[5m])
```

### ARC Controller Metrics (if exposed)

```promql
# Controller-specific metrics (if your ARC version exposes them)
gha_controller_queue_depth
gha_controller_reconcile_duration_seconds
gha_runner_scale_set_runners
```

### Resource Usage Dashboard Queries

Create a dashboard with these panels:

**Panel 1: CPU Usage per Runner**
```promql
sum(rate(container_cpu_usage_seconds_total{namespace="arc-runners", container!=""}[5m])) by (pod)
```

**Panel 2: Memory Usage per Runner**
```promql
sum(container_memory_working_set_bytes{namespace="arc-runners", container!=""}) by (pod)
```

**Panel 3: Active Runners**
```promql
count(kube_pod_status_phase{namespace="arc-runners", phase="Running"})
```

**Panel 4: Pod Status Distribution**
```promql
sum(kube_pod_status_phase{namespace="arc-runners"}) by (phase)
```

**Panel 5: Network Throughput**
```promql
sum(rate(container_network_receive_bytes_total{namespace="arc-runners"}[5m])) by (pod)
sum(rate(container_network_transmit_bytes_total{namespace="arc-runners"}[5m])) by (pod)
```

## Advanced: Custom Metrics from Runners

If you want to export custom metrics from your GitHub Actions workflows:

### Option 1: Prometheus Pushgateway

```yaml
# In your GitHub Actions workflow
- name: Push metrics
  run: |
    echo "job_duration_seconds 123.45" | curl --data-binary @- \
      http://lgtm-mimir-nginx.monitoring/api/v1/push
```

### Option 2: OpenTelemetry Collector

Send job telemetry through the OpenTelemetry Collector:

```yaml
# In your runner pod, add OTEL environment variables
env:
- name: OTEL_EXPORTER_OTLP_ENDPOINT
  value: "http://otel-collector.monitoring:4318"
- name: OTEL_SERVICE_NAME
  value: "github-runner"
```

Then instrument your workflows to send traces/metrics via OTLP.

## Alerting

Create alerts in Grafana for common issues:

### High Memory Usage
```promql
(container_memory_usage_bytes{namespace="arc-runners"} /
 container_spec_memory_limit_bytes{namespace="arc-runners"}) > 0.9
```

### Pod Crash Loop
```promql
rate(kube_pod_container_status_restarts_total{namespace="arc-runners"}[15m]) > 0
```

### No Available Runners
```promql
count(kube_pod_status_phase{namespace="arc-runners", phase="Running"}) == 0
```

### High CPU Usage
```promql
rate(container_cpu_usage_seconds_total{namespace="arc-runners"}[5m]) > 1.5
```

## Troubleshooting

### Logs not appearing in Loki

1. Ensure `ACTIONS_RUNNER_PRINT_LOG_TO_STDOUT=1` is set in runner pods
2. Check Promtail is scraping the arc-runners namespace:
   ```bash
   kubectl logs -n monitoring -l app.kubernetes.io/name=promtail
   ```

### Metrics not appearing in Prometheus

1. Check PodMonitor is in the correct namespace
2. Verify the `release: prometheus-operator` label is present
3. Check Prometheus targets: `kubectl port-forward -n monitoring svc/prometheus-operator-kube-prom-prometheus 9090:9090`

### Runner pods not visible

1. Ensure kube-state-metrics is running:
   ```bash
   kubectl get pods -n monitoring -l app.kubernetes.io/name=kube-state-metrics
   ```

## Best Practices

1. **Resource Limits**: Always set CPU/memory limits on runner pods for accurate monitoring
2. **Labels**: Use consistent labels across runner scale sets for easier filtering
3. **Retention**: Configure appropriate log/metric retention based on your compliance needs
4. **Dashboards**: Create separate dashboards for different runner scale sets or organizations
5. **Alerts**: Set up alerts for pod crashes, high resource usage, and job failures

## Additional Resources

- [ARC Documentation](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners-with-actions-runner-controller)
- [Prometheus Operator CRDs](https://github.com/prometheus-operator/prometheus-operator/blob/main/Documentation/user-guides/getting-started.md)
- [Grafana Dashboards](https://grafana.com/grafana/dashboards/)
