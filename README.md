# ML Platform Helmfile

Automated installation of **Istio**, **Knative Serving**, **KServe**, and custom **ClusterServingRuntime** using Helmfile.

## Overview

This Helmfile configuration deploys a complete ML serving platform with:

| Component | Version | Description |
|-----------|---------|-------------|
| Istio | 1.25.1 | Service mesh for traffic management and mTLS |
| Knative Serving | 1.17.2 | Serverless runtime with scale-to-zero |
| KServe | 0.15.0 | ML model serving with inference protocol |
| ComfyServer | 0.2 | Custom runtime for ComfyUI workflows |

> [!TIP]
> **ComfyServer Images:** Pre-built images are available from [MJZLOTR/ComfyServer](https://github.com/MJZLOTR/ComfyServer/tree/master).


## Prerequisites

- **Kubernetes cluster** (v1.28+)
- **kubectl** configured with cluster access
- **Helm** v3.12+
- **Helmfile** v1.0+

### Optional (for production)

- **cert-manager** for TLS certificate automation
- **NVIDIA GPU Operator** for GPU workloads
- **Prometheus** for observability

## Quick Start

### 1. Install Dependencies

```bash
# Install Helm
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Install Helmfile
curl -fsSL https://github.com/helmfile/helmfile/releases/latest/download/helmfile_linux_amd64.tar.gz | \
    tar xz && sudo mv helmfile /usr/local/bin/
```

### 2. Install Platform

```bash
# Default environment (kind/local cluster)
helmfile -e default sync

# Production environment
helmfile -e production sync

# Dry-run (template only)
helmfile -e default template
```

### 3. Uninstall

```bash
helmfile -e default destroy
```

## Configuration

### Environment Files

| File | Description |
|------|-------------|
| `environments/default.yaml` | Local/development settings (kind cluster) |
| `environments/production.yaml` | Production settings (OTC CCE with ELB, TLS) |

### Key Configuration Options

#### Istio Ingress Gateway (production)

```yaml
istio:
  ingressGateway:
    serviceType: LoadBalancer
    elb:
      enabled: true
      id: "your-elb-id"        # OTC ELB instance ID
      class: performance
      algorithm: ROUND_ROBIN
```

#### Knative Serving

```yaml
knativeServing:
  domain: "knative.example.com"  # Base domain for services
  timeouts:
    revisionTimeout: 1200        # 20 min for long inference
    maxRevisionTimeout: 2400     # 40 min max
  tls:
    enabled: true
    clusterIssuer: "letsencrypt-prod"
```

#### KServe

```yaml
kserve:
  deploymentMode: Serverless
  storageInitializer:
    enableDirectPvcVolumeMount: true  # Fast PVC mounting
```

#### ComfyServer Runtime

```yaml
comfyServer:
  enabled: true
  registry: "swr.eu-de.otc.t-systems.com/your-registry"
  image: "comfyserver:0.2-slim"
  resources:
    requests:
      gpu: "1"
```

## Directory Structure

```
ml-platform-helmfile/
├── helmfile.yaml              # Main Helmfile
├── environments/
│   ├── default.yaml           # Local cluster values
│   └── production.yaml        # Production values
├── values/
│   ├── istiod.yaml.gotmpl
│   ├── istio-gateway.yaml.gotmpl
│   ├── knative.yaml.gotmpl
│   ├── kserve.yaml.gotmpl
│   ├── comfyserver.yaml.gotmpl
│   └── inference-namespace.yaml.gotmpl
├── charts/
│   ├── knative-config/        # KnativeServing CR (manual chart)
│   ├── serving-runtime/       # ClusterServingRuntime
│   └── inference-namespace/   # S3 credentials and namespace
└── README.md
```

> [!NOTE]
> **Why `knative-config` is a manual chart:** The official Knative Operator Helm chart does not support applying custom ConfigMaps (e.g., `config-domain`, `config-defaults`, `config-features`) or the `KnativeServing` CR with environment-specific configurations. Therefore, a manual chart was created to deploy the `KnativeServing` CR with the required ConfigMap settings embedded in the CR's `spec.config` field.

## Helmfile Commands

```bash
# Show installation order (DAG)
helmfile show-dag

# Preview changes
helmfile -e production diff

# Apply changes
helmfile -e production apply

# Install specific layer
helmfile -e production -l layer=istio apply

# Destroy all
helmfile -e production destroy
```

## Deployed ConfigMaps

### Knative Serving (namespace: `knative-serving`)

| ConfigMap | Purpose |
|-----------|---------|
| `config-domain` | Base domain for Knative Services |
| `config-defaults` | Timeout settings (1200s/2400s for inference) |
| `config-network` | External domain TLS enabled |
| `config-features` | podspec-nodeselector, podspec-tolerations |
| `config-certmanager` | ClusterIssuer for TLS certificates |
| `config-observability` | Prometheus metrics backend |

### KServe (namespace: `kserve`)

| ConfigMap | Purpose |
|-----------|---------|
| `inferenceservice-config` | Storage initializer, ingress gateway config |

### Inference Namespace (namespace: configurable, default: `inference`)

| Resource | Purpose |
|----------|---------|
| Namespace | Labeled with `istio-injection=enabled` |
| Secret (`s3-secret`) | S3 credentials for model storage |
| ServiceAccount | SA with S3 secret annotation for KServe |

## Deploying an InferenceService

After platform installation, deploy a ComfyUI workflow. For detailed information about ComfyServer, workflow configuration, and API usage, see the [ComfyServer documentation](https://github.com/MJZLOTR/ComfyServer).

### Example InferenceService (Simple Workflow)

The following example deploys the **simple workflow** documented in [ComfyServer](https://github.com/MJZLOTR/ComfyServer). This workflow demonstrates basic text-to-image generation:

```yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: simple-workflow
  namespace: inference
  annotations:
    autoscaling.knative.dev/scale-to-zero-pod-retention-period: "10m"
    autoscaling.knative.dev/scale-down-delay: "5m"
    storage.kserve.io/readonly: "false"
    autoscaling.knative.dev/panic-threshold-percentage: "200.0"
    autoscaling.knative.dev/panic-window-percentage: "10.0"
    autoscaling.knative.dev/target: "5"
    autoscaling.knative.dev/target-utilization-percentage: "70"
    autoscaling.knative.dev/window: 60s
    serving.knative.dev/progress-deadline: 10m
    autoscaling.knative.dev/target-burst-capacity: "-1"
    autoscaling.knative.dev/metric: concurrency
spec:
  predictor:
    minReplicas: 0
    maxReplicas: 15
    timeout: 300
    serviceAccountName: sa-s3
    tolerations:
      - key: "nvidia.com/gpu"
        operator: "Equal"
        value: "present"
        effect: "NoSchedule"
    model:
      modelFormat:
        name: comfy
      protocolVersion: v2
      storageUri: "s3://comfy/simple_workflow"
      args:
        - --models_path_config
        - "/mnt/models/config.yaml"
        - --workflow
        - "/mnt/models/workflows/simple_wf.json"
        - --disable_save_nodes
        - "true"
      resources:
        limits:
          cpu: "2"
          memory: 8Gi
          nvidia.com/gpu: "1"
        requests:
          cpu: "1"
          memory: 4Gi
          nvidia.com/gpu: "1"
```

> [!IMPORTANT]
> The `storageUri` points to the OBS/S3 bucket location containing your models, workflows, and configuration files. This follows the same directory structure used by [ComfyServer in container mode](https://github.com/MJZLOTR/ComfyServer)—the contents are mounted to `/mnt/models` and referenced via the `--models_path_config` and `--workflow` arguments.

### Sending Image Generation Requests

Once your InferenceService is deployed, send requests using the KServe V2 inference protocol:

```bash
# Create request payload
cat > request.json << 'EOF'
{
  "id": "1",
  "inputs": [
    {"name": "6_text", "datatype": "BYTES", "shape": [1], "data": ["a futuristic city at sunset, detailed, cyberpunk"]}
  ],
  "outputs": [
    {
      "name": "8_0_IMAGE",
      "datatype": "FP32",
      "shape": [-1],
      "parameters": {
        "rename": "image",
        "binary_data": false,
        "to_base64": true
      }
    }
  ]
}
EOF

# Send request and decode image
curl -H "Content-Type: application/json" \
     https://<service-name>-predictor.<namespace>.<domain>/v2/models/<service-name>/infer \
     -d @request.json | jq -r '.outputs[0].data[0]' | base64 -d > generated_image.png
```

> [!NOTE]
> The input names (e.g., `3_steps`, `6_text`) and output names (e.g., `8_0_IMAGE`) correspond to node IDs in your ComfyUI workflow. See [ComfyServer](https://github.com/MJZLOTR/ComfyServer) for details on workflow parameterization.
