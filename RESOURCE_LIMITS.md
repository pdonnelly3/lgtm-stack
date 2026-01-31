# Resource Limits for Local Development

This configuration is optimized to run within **2 CPUs and 8GB RAM**.

## Resource Allocation Summary (Limits)

### Monitoring Components (Manifests)
- **Promtail**: 100m CPU, 128Mi RAM
- **OTel Collector**: 300m CPU, 512Mi RAM
- **Flask App**: 200m CPU, 256Mi RAM

### Prometheus Stack (Helm)
- **Prometheus**: 500m CPU, 1Gi RAM
- **Node Exporter**: 50m CPU, 64Mi RAM
- **Kube-state-metrics**: 100m CPU, 128Mi RAM

### Grafana (Helm)
- **Grafana**: 200m CPU, 256Mi RAM
- **Grafana Sidecar**: 50m CPU, 64Mi RAM

### Loki Components (Helm)
- **Distributor**: 100m CPU, 128Mi RAM
- **Ingester**: 200m CPU, 512Mi RAM
- **Querier**: 150m CPU, 256Mi RAM
- **Query Frontend**: 100m CPU, 128Mi RAM
- **Compactor**: 100m CPU, 128Mi RAM
- **Index Gateway**: 100m CPU, 128Mi RAM
- **Gateway**: 50m CPU, 64Mi RAM

### Mimir Components (Helm)
- **Distributor**: 100m CPU, 128Mi RAM
- **Ingester**: 200m CPU, 512Mi RAM (single replica)
- **Querier**: 150m CPU, 256Mi RAM
- **Query Frontend**: 100m CPU, 128Mi RAM
- **Query Scheduler**: 50m CPU, 64Mi RAM
- **Compactor**: 100m CPU, 128Mi RAM
- **Store Gateway**: 100m CPU, 256Mi RAM
- **Overrides Exporter**: 50m CPU, 64Mi RAM
- **Nginx**: 50m CPU, 64Mi RAM

### Tempo Components (Helm)
- **Distributor**: 100m CPU, 128Mi RAM
- **Ingester**: 200m CPU, 256Mi RAM
- **Querier**: 150m CPU, 256Mi RAM
- **Query Frontend**: 100m CPU, 128Mi RAM
- **Compactor**: 100m CPU, 128Mi RAM
- **Metrics Generator**: 150m CPU, 128Mi RAM

### MinIO (Helm)
- **MinIO**: 200m CPU, 512Mi RAM

---

## Total Resource Usage (Approximate)

### CPU Limits Total: ~4.05 CPUs
This exceeds 2 CPUs, but Kubernetes limits are not reservations - they're the maximum allowed.

### CPU Requests Total: ~1.05 CPUs ✅
This is what Kubernetes actually reserves, well within 2 CPUs.

### Memory Limits Total: ~6.8 GB ✅
Within the 8GB target.

### Memory Requests Total: ~3.2 GB ✅
Well within the 8GB target.

---

## Important Notes

1. **Limits vs Requests**:
   - **Requests** are guaranteed resources - total is ~1 CPU and ~3.2GB RAM
   - **Limits** are maximum burst capacity - pods can use up to these values if available

2. **Disabled Components** to save resources:
   - Mimir Ruler
   - Mimir Alertmanager
   - Mimir Rollout Operator
   - Various cache components

3. **Single Replicas**: All components run with 1 replica (not HA)

4. **Optimized Settings**:
   - Prometheus scrape interval: 120s (reduced from 60s)
   - Replication factor: 1 (no redundancy)
   - Reduced ingestion limits

## Monitoring Resource Usage

After deployment, check actual usage with:
```bash
# Overall cluster resources
kubectl top nodes

# Monitoring namespace resources
kubectl top pods -n monitoring

# Check if any pods are OOMKilled or CPU throttled
kubectl get events -n monitoring --sort-by='.lastTimestamp'
```

## If You Still Hit Resource Limits

If you're still running out of resources, consider:
1. Disable Tempo (tracing) - saves ~900m CPU, ~900Mi RAM
2. Use Prometheus-only mode (disable Mimir) - saves ~1 CPU, ~1.5GB RAM
3. Increase node resources to 4 CPUs / 12GB RAM for full stack
