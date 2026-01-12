# Elastic Agent to Logstash in Kubernetes

Complete example of Elastic Agent sending data to Logstash in Kubernetes, with Elasticsearch output using Fleet-compatible data streams.

## Architecture

```
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│   Sample App     │     │  Elastic Agent   │     │    Logstash      │
│   (Pods/Logs)    │────▶│  (Deployment)    │────▶│  (Deployment)    │
└──────────────────┘     └──────────────────┘     └────────┬─────────┘
                              Port 5044                     │
                                                            ▼
                                                 ┌──────────────────┐
                                                 │  Elasticsearch   │
                                                 │  (Data Streams)  │
                                                 └──────────────────┘
```

## Features

- ✅ Logstash with Beats input (port 5044) for Fleet/Elastic Agent
- ✅ Elasticsearch output with username/password authentication
- ✅ SSL verification disabled for self-signed certificates
- ✅ Persistent queue (4GB) for data durability
- ✅ Fleet-compatible data stream format
- ✅ Elastic Agent as Deployment (not DaemonSet)
- ✅ Kubernetes metadata enrichment
- ✅ Sample application for testing

## Prerequisites

- Kubernetes cluster (local or cloud)
- `kubectl` configured to access your cluster
- Elasticsearch cluster accessible from Kubernetes

## Quick Start

### 1. Update Elasticsearch Credentials

Edit `elasticsearch-secret.yaml` with your Elasticsearch details:

```yaml
stringData:
  ELASTICSEARCH_HOST: "https://your-elasticsearch:9200"
  ELASTICSEARCH_USERNAME: "elastic"
  ELASTICSEARCH_PASSWORD: "your-password"
```

### 2. Deploy Everything

```bash
# Apply in order
kubectl apply -f elasticsearch-secret.yaml
kubectl apply -f logstash-config.yaml
kubectl apply -f logstash-pvc.yaml
kubectl apply -f logstash-deployment.yaml
kubectl apply -f logstash-service.yaml
kubectl apply -f elastic-agent-rbac.yaml
kubectl apply -f elastic-agent-config.yaml
kubectl apply -f elastic-agent-deployment.yaml
kubectl apply -f sample-app-deployment.yaml
```

Or deploy all at once:

```bash
kubectl apply -f .
```

### 3. Verify Deployment

```bash
# Check all pods are running
kubectl get pods

# Check Logstash logs
kubectl logs -l app=logstash --tail=50

# Check Elastic Agent logs
kubectl logs -l app=elastic-agent --tail=50

# Check sample app logs
kubectl logs -l app=sample-app --tail=20
```

## Fleet Output Configuration

To use Logstash as an output in Fleet:

1. **Internal Cluster Access**: Use `logstash:5044`
2. **External Access**: Change service type in `logstash-service.yaml`:
   - `LoadBalancer` for cloud providers
   - `NodePort` for on-premises

In Fleet UI → Settings → Outputs:
- **Type**: Logstash
- **Hosts**: `logstash:5044` (or external IP/hostname)
- **SSL**: Disabled (as configured)

## Files Overview

| File | Description |
|------|-------------|
| `elasticsearch-secret.yaml` | Elasticsearch credentials |
| `logstash-config.yaml` | Logstash pipeline configuration |
| `logstash-pvc.yaml` | Persistent volume for queue |
| `logstash-deployment.yaml` | Logstash deployment |
| `logstash-service.yaml` | Service for Fleet output (port 5044) |
| `elastic-agent-rbac.yaml` | RBAC for Elastic Agent |
| `elastic-agent-config.yaml` | Elastic Agent configuration |
| `elastic-agent-deployment.yaml` | Elastic Agent deployment |
| `sample-app-deployment.yaml` | Sample log generator |

## Persistent Queue Settings

Optimized configuration in `logstash-config.yaml`:

```yaml
queue.type: persisted
queue.max_bytes: 4gb              # Maximum queue size
queue.checkpoint.writes: 1024     # Checkpoint every 1024 events
queue.checkpoint.acks: 1024       # Checkpoint every 1024 acks
queue.checkpoint.interval: 1000   # Checkpoint timeout (ms)
queue.drain: true                 # Drain queue before shutdown
```

**Why these settings?**
- **4GB max_bytes**: Sufficient buffer for ~5-10 million events
- **checkpoint.writes: 1024**: Balance between durability and performance
- **drain: true**: Prevents data loss during restarts

## Customization

### Enable SSL on Logstash Input

In `logstash-config.yaml`, uncomment SSL settings:

```ruby
beats {
  port => 5044
  ssl => true
  ssl_certificate => "/etc/logstash/certs/logstash.crt"
  ssl_key => "/etc/logstash/certs/logstash.key"
}
```

### Change Data Stream Naming

Modify in `logstash-config.yaml`:

```ruby
data_stream_type => "logs"
data_stream_dataset => "your-dataset"
data_stream_namespace => "your-namespace"
```

### Scale Logstash

Increase replicas (note: each replica needs its own PVC):

```yaml
spec:
  replicas: 3
```

## Troubleshooting

### Logstash not starting
```bash
kubectl describe pod -l app=logstash
kubectl logs -l app=logstash --previous
```

### Elastic Agent connection refused
- Verify Logstash service: `kubectl get svc logstash`
- Check Logstash is listening: `kubectl exec -it <logstash-pod> -- ss -tlnp`

### No data in Elasticsearch
- Check Logstash output logs for errors
- Verify Elasticsearch credentials in the secret
- Test Elasticsearch connectivity from Logstash pod

## Cleanup

```bash
kubectl delete -f .
```
