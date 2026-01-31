# Resource Limits for Local Development

This configuration is optimized to run within **4 CPUs and 8GB RAM** at peak load, with guaranteed **2 CPUs and 4GB RAM** reserved.

## Resource Allocation Summary

### Monitoring Components (Manifests)
| Component | CPU Request | CPU Limit | Memory Request | Memory Limit |
|-----------|-------------|-----------|----------------|--------------|
| **Promtail** | 30m | 100m | 64Mi | 128Mi |
| **OTel Collector** | 150m | 350m | 384Mi | 768Mi |

### Prometheus Stack (Helm)
| Component | CPU Request | CPU Limit | Memory Request | Memory Limit |
|-----------|-------------|-----------|----------------|--------------|
| **Prometheus** | 300m | 600m | 1024Mi | 1792Mi |

### Grafana (Helm)
| Component | CPU Request | CPU Limit | Memory Request | Memory Limit |
|-----------|-------------|-----------|----------------|--------------|
| **Grafana** | 100m | 200m | 160Mi | 256Mi |
| **Grafana Sidecar** | 30m | 50m | 32Mi | 64Mi |

### Loki Components (Helm)
| Component | CPU Request | CPU Limit | Memory Request | Memory Limit |
|-----------|-------------|-----------|----------------|--------------|
| **Distributor** | 60m | 100m | 64Mi | 128Mi |
| **Gateway** | 25m | 50m | 32Mi | 64Mi |
| **Ingester** | 100m | 200m | 192Mi | 640Mi |
| **Querier** | 80m | 150m | 160Mi | 256Mi |
| **Query Frontend** | 50m | 100m | 64Mi | 128Mi |
| **Compactor** | 40m | 100m | 64Mi | 128Mi |
| **Index Gateway** | 40m | 100m | 64Mi | 128Mi |

### Mimir Components (Helm)
| Component | CPU Request | CPU Limit | Memory Request | Memory Limit |
|-----------|-------------|-----------|----------------|--------------|
| **Distributor** | 60m | 100m | 64Mi | 128Mi |
| **Ingester** | 150m | 200m | 384Mi | 640Mi |
| **Querier** | 100m | 150m | 160Mi | 256Mi |
| **Query Frontend** | 50m | 100m | 64Mi | 128Mi |
| **Query Scheduler** | 25m | 50m | 32Mi | 64Mi |
| **Compactor** | 40m | 100m | 64Mi | 128Mi |
| **Store Gateway** | 40m | 100m | 128Mi | 256Mi |
| **Overrides Exporter** | 25m | 50m | 32Mi | 64Mi |
| **Nginx** | 25m | 50m | 32Mi | 64Mi |

### Tempo Components (Helm)
| Component | CPU Request | CPU Limit | Memory Request | Memory Limit |
|-----------|-------------|-----------|----------------|--------------|
| **Distributor** | 60m | 100m | 64Mi | 128Mi |
| **Ingester** | 100m | 200m | 192Mi | 384Mi |
| **Querier** | 80m | 130m | 160Mi | 256Mi |
| **Query Frontend** | 50m | 100m | 64Mi | 128Mi |
| **Compactor** | 40m | 100m | 64Mi | 128Mi |
| **Metrics Generator** | 60m | 120m | 64Mi | 128Mi |

### MinIO (Helm)
| Component | CPU Request | CPU Limit | Memory Request | Memory Limit |
|-----------|-------------|-----------|----------------|--------------|
| **MinIO** | 150m | 250m | 384Mi | 896Mi |

---

## Total Resource Usage

### Summary Table
| Resource Type | Requests (Guaranteed) | Limits (Max Burst) |
|---------------|----------------------|--------------------|
| **CPU** | **2.0 CPUs** ✅ | **4.0 CPUs** ✅ |
| **Memory** | **4.0 GB** ✅ | **8.0 GB** ✅ |

### Breakdown
- **CPU Requests**: 2000m (2.0 CPUs) - guaranteed by Kubernetes scheduler
- **CPU Limits**: 4000m (4.0 CPUs) - maximum burst capacity
- **Memory Requests**: 4096Mi (4.0 GB) - guaranteed memory allocation
- **Memory Limits**: 8192Mi (8.0 GB) - maximum memory before OOMKill

---

## Important Notes

1. **Limits vs Requests**:
   - **Requests** (2 CPU / 4GB RAM) are guaranteed resources that Kubernetes reserves on nodes
   - **Limits** (4 CPU / 8GB RAM) are maximum burst capacity - pods can use up to these values if available
   - CPU is throttled when exceeding limits; memory triggers OOMKill

2. **Disabled Components** to save resources:
   - Mimir Ruler
   - Mimir Alertmanager
   - Mimir Rollout Operator
   - Various cache components (memcached)
   - Kubernetes control plane monitoring (scheduler, controller-manager, etcd, kube-proxy)

3. **Single Replicas**: All components run with 1 replica (not HA)

4. **Optimized Settings**:
   - Prometheus scrape interval: 120s (reduced from 60s)
   - Replication factor: 1 (no redundancy)
   - Reduced ingestion limits
   - MinIO single instance

5. **Resource Distribution**:
   - **Highest allocation**: Prometheus (critical for metrics), Mimir Ingester, MinIO (storage backend)
   - **Medium allocation**: Loki/Mimir/Tempo Ingesters and Queriers (data processing)
   - **Lower allocation**: Distributors, Frontends, Gateways (routing/proxying)

## Monitoring Resource Usage

After deployment, check actual usage with:
```bash
# Overall cluster resources
kubectl top nodes

# Monitoring namespace resources
kubectl top pods -n monitoring --sort-by=cpu
kubectl top pods -n monitoring --sort-by=memory

# Check if any pods are OOMKilled or CPU throttled
kubectl get events -n monitoring --sort-by='.lastTimestamp' | grep -i "oom\|throttl"

# Watch resource usage in real-time
watch kubectl top pods -n monitoring
```

## Performance Expectations

With this configuration:
- ✅ **Light workloads**: 1-2 apps, low traffic - runs smoothly
- ✅ **Moderate workloads**: 5-10 apps, moderate traffic - acceptable performance
- ⚠️ **Heavy workloads**: 20+ apps, high traffic - expect CPU throttling and slower queries
- ❌ **Production workloads**: Not suitable for production; increase to 8+ CPUs / 16+ GB RAM

## If You Hit Resource Limits

If experiencing performance issues:

1. **Quick wins** (disable optional components):
   ```bash
   # Disable Tempo (tracing) - saves ~390m CPU, ~1GB RAM
   helm upgrade lgtm -n monitoring --set tempo.enabled=false -f helm/values-lgtm.local.yaml

   # Disable Grafana dashboards sidecar - saves ~30m CPU, ~32Mi RAM
   helm upgrade lgtm -n monitoring --set grafana.sidecar.dashboards.enabled=false -f helm/values-lgtm.local.yaml
   ```

2. **Increase node resources**:
   - **Recommended for testing**: 4 CPUs / 8GB RAM (current config)
   - **Recommended for light prod**: 8 CPUs / 16GB RAM
   - **Recommended for production**: 16 CPUs / 32GB RAM

3. **Use Prometheus-only mode** (disable LGTM, keep just Prometheus + Grafana):
   - Saves ~2.5 CPUs and ~4GB RAM
   - Loses logs (Loki) and traces (Tempo)

4. **Optimize query patterns**:
   - Reduce Grafana dashboard refresh rates
   - Increase Prometheus scrape interval to 180s or 300s
   - Use recording rules for frequently queried metrics
