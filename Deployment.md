# LGTM Stack Deployment Guide

This guide explains how to deploy the LGTM (Loki, Grafana, Tempo, Mimir) stack using Helm charts and connect your applications with OpenTelemetry instrumentation.

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Deployment Options](#deployment-options)
4. [Replacing the Flask App](#replacing-the-flask-app)
5. [Connecting Your Application with OpenTelemetry](#connecting-your-application-with-opentelemetry)
6. [Configuration Examples](#configuration-examples)
7. [Troubleshooting](#troubleshooting)

## Overview

The LGTM stack provides a complete observability solution with:
- **Loki**: Log aggregation and querying
- **Grafana**: Visualization and dashboards
- **Tempo**: Distributed tracing
- **Mimir**: Metrics storage and querying
- **OpenTelemetry Collector**: Telemetry data collection and routing

## Prerequisites

### Required Tools
- `kubectl` (configured with your cluster)
- `helm` (v3+)
- `make` (optional, for using the provided Makefile)

## Deployment Options

### Local Development (Minikube/Docker Desktop)

```bash
# Install the LGTM stack locally
make install-local

# Or manually:
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

kubectl create ns monitoring
helm install prometheus-operator --version 75.9.0 -n monitoring \
  prometheus-community/kube-prometheus-stack -f helm/values-prometheus.yaml
helm install lgtm --version 2.1.0 -n monitoring \
  grafana/lgtm-distributed -f helm/values-lgtm.local.yaml
```

### Generic Kubernetes Deployment

```bash
# Install the LGTM stack on any Kubernetes cluster
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

kubectl create ns monitoring
helm install prometheus-operator --version 75.9.0 -n monitoring \
  prometheus-community/kube-prometheus-stack -f helm/values-prometheus.yaml
helm install lgtm --version 2.1.0 -n monitoring \
  grafana/lgtm-distributed -f helm/values-lgtm.local.yaml
```

## Replacing the Flask App

The current setup includes a sample Flask application. To replace it with your own application:

### 1. Remove the Flask App

```bash
# Remove the existing flask-app deployment
kubectl delete -f manifests/app/flask-app.yaml
```

### 2. Deploy Your Application

Create a deployment for your application following this template:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: your-app
  labels:
    app: your-app
spec:
  selector:
    matchLabels:
      app: your-app
  replicas: 1
  template:
    metadata:
      labels:
        app: your-app
    spec:
      containers:
      - name: your-app
        image: your-registry/your-app:latest
        env:
        - name: OTEL_EXPORTER_OTLP_ENDPOINT
          value: "http://otel-collector:4318"
        - name: OTEL_SERVICE_NAME
          value: "your-app"
        - name: OTEL_RESOURCE_ATTRIBUTES
          value: "service.name=your-app,service.version=1.0.0"
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: your-app
spec:
  selector:
    app: your-app
  ports:
  - protocol: TCP
    port: 8080
    targetPort: 8080
```

### 3. Deploy the OpenTelemetry Collector

The OpenTelemetry Collector is already configured to route telemetry data to the LGTM stack:

```bash
kubectl apply -f manifests/otel-collector.yaml
```

## Connecting Your Application with OpenTelemetry

### 1. Instrument Your Application

Add OpenTelemetry instrumentation to your Python application:

```python
from opentelemetry import trace
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.instrumentation.flask import FlaskInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor

# Configure OpenTelemetry
trace.set_tracer_provider(TracerProvider())
trace.get_tracer_provider().add_span_processor(
    BatchSpanProcessor(OTLPSpanExporter(endpoint="http://otel-collector:4318/v1/traces"))
)

# Instrument Flask
FlaskInstrumentor().instrument()
RequestsInstrumentor().instrument()
```

#### Node.js (Express)

```javascript
const { NodeTracerProvider } = require('@opentelemetry/sdk-trace-node');
const { OTLPTraceExporter } = require('@opentelemetry/exporter-otlp-http');
const { BatchSpanProcessor } = require('@opentelemetry/sdk-trace-base');
const { registerInstrumentations } = require('@opentelemetry/instrumentation');
const { ExpressInstrumentation } = require('@opentelemetry/instrumentation-express');

const provider = new NodeTracerProvider();
const exporter = new OTLPTraceExporter({
  url: 'http://otel-collector:4318/v1/traces',
});
provider.addSpanProcessor(new BatchSpanProcessor(exporter));
provider.register();

registerInstrumentations({
  instrumentations: [new ExpressInstrumentation()],
});
```



### 2. Environment Variables

Set these environment variables in your application deployment:

```yaml
env:
- name: OTEL_EXPORTER_OTLP_ENDPOINT
  value: "http://otel-collector:4318"
- name: OTEL_SERVICE_NAME
  value: "your-app"
- name: OTEL_RESOURCE_ATTRIBUTES
  value: "service.name=your-app,service.version=1.0.0,deployment.environment=production"
- name: OTEL_TRACES_SAMPLER
  value: "always_on"
- name: OTEL_METRICS_EXPORTER
  value: "otlp"
- name: OTEL_LOGS_EXPORTER
  value: "otlp"
```

## Configuration Examples

### Customizing Helm Values

You can customize the LGTM stack by modifying the values files:

#### Local Development (`helm/values-lgtm.local.yaml`)

```yaml
grafana:
  enabled: true
  # Customize Grafana settings
  persistence:
    enabled: true
    size: 10Gi

tempo:
  enabled: true
  # Configure trace retention
  storage:
    trace:
      backend: s3
      s3:
        bucket: tempo
        prefix: traces
```

#### Production Customization

For production deployments, you can create your own values file based on `helm/values-lgtm.local.yaml`:

```yaml
grafana:
  enabled: true
  # Configure ingress for external access
  ingress:
    enabled: true
    hosts: ["grafana.yourdomain.com"]
  persistence:
    enabled: true
    size: 50Gi

tempo:
  enabled: true
  # Configure your preferred storage backend
  storage:
    trace:
      backend: s3  # or gcs, azure, etc.
      s3:
        bucket: your-tempo-bucket
        prefix: traces
```

### Adding Custom Dashboards

Create custom Grafana dashboards by adding them to the `manifests/` directory:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: custom-dashboard
  labels:
    grafana_dashboard: "1"
data:
  dashboard.json: |
    {
      "title": "Custom Dashboard",
      "panels": [...]
    }
```

## Troubleshooting

### Common Issues

#### 1. OpenTelemetry Collector Not Receiving Data

Check if the collector is running:
```bash
kubectl get pods -l app=otel-collector
kubectl logs -l app=otel-collector
```

Verify the endpoint configuration:
```bash
kubectl get svc otel-collector
```

#### 2. Traces Not Appearing in Grafana

Check Tempo configuration:
```bash
kubectl get pods -l app.kubernetes.io/name=tempo
kubectl logs -l app.kubernetes.io/name=tempo -c tempo
```

#### 3. Metrics Not Showing in Grafana

Check Mimir configuration:
```bash
kubectl get pods -l app.kubernetes.io/name=mimir
kubectl logs -l app.kubernetes.io/name=mimir -c mimir
```

### Useful Commands

```bash
# Get Grafana password
make get-grafana-password

# Port forward to access Grafana
kubectl port-forward svc/lgtm-grafana 3000:80 -n monitoring

# Check all LGTM components
kubectl get pods -n monitoring

# View logs for specific components
kubectl logs -f deployment/lgtm-tempo-distributor -n monitoring
kubectl logs -f deployment/lgtm-mimir-distributor -n monitoring
kubectl logs -f deployment/lgtm-loki-distributor -n monitoring
```

### Uninstalling

```bash
# Uninstall the entire stack
make uninstall

# Or manually:
helm uninstall lgtm -n monitoring
helm uninstall prometheus-operator -n monitoring
kubectl delete -f manifests/promtail.yaml || true
kubectl delete -f manifests/otel-collector.yaml || true
kubectl delete ns monitoring || true
```

## Next Steps

1. **Customize Dashboards**: Create custom Grafana dashboards for your application metrics
2. **Set Up Alerts**: Configure Prometheus rules and Grafana alerts
3. **Scale Components**: Adjust resource limits and replicas based on your workload
4. **Security**: Configure RBAC, network policies, and TLS for production use
5. **Backup**: Set up backup strategies for your telemetry data

For more information, refer to the official documentation:
- [Grafana LGTM Stack](https://grafana.com/docs/lgtm/)
- [OpenTelemetry](https://opentelemetry.io/docs/)
- [Helm Charts](https://helm.sh/docs/) 