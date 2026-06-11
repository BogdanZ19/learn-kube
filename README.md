# Kubernetes Manifests for SynergyChat

This repository contains Kubernetes manifest files for deploying the SynergyChat application and supporting services. It includes API, web, crawler, gateway, and resource limit examples for a cluster-based environment.

## Repository Structure

- `api-*`: manifests for the SynergyChat API service
- `web-*`: manifests for the SynergyChat web frontend
- `crawler-*`: manifests for the crawler service and its namespace
- `app-gateway*`: Gateway API resources for routing HTTP traffic
- `resource_limits/`: sample deployments and autoscaling manifests demonstrating CPU and memory limits

## Components

### API Service
- `api-deployment.yaml`: Deployment for `synergychat-api`
- `api-service.yaml`: Service exposing the API on port `80`
- `api-configmap.yaml`: ConfigMap with API environment settings
- `api-pvc.yaml`: PersistentVolumeClaim for API storage
- `api-httproute.yaml`: (not referenced in sample manifests) may define HTTP routing for the API

### Web Frontend
- `web-deployment.yaml`: Deployment for `synergychat-web`
- `web-service.yaml`: Service exposing the web frontend on port `80`
- `web-configmap.yaml`: ConfigMap with frontend environment settings
- `web-hpa.yaml`: HorizontalPodAutoscaler targeting the web deployment
- `web-httproute.yaml`: HTTPRoute to route incoming requests to the `web-service`

### Crawler Service
- `crawler-deployment.yaml`: Multi-container deployment for `synergychat-crawler` in the `crawler` namespace
- `crawler-service.yaml`: Service exposing the crawler on port `80`
- `crawler-configmap.yaml`: ConfigMap with crawler ports, keywords, and storage path

### Gateway
- `app-gatewayclass.yaml`: Defines a GatewayClass using `gateway.envoyproxy.io/gatewayclass-controller`
- `app-gateway.yaml`: Gateway listening on HTTP port `80`

### Resource Limit Examples
- `resource_limits/testcpu-deployment.yaml`: Deployment illustrating CPU limits
- `resource_limits/testcpu-hpa.yaml`: HPA targeting the CPU example deployment
- `resource_limits/testram-*`: Example manifests showing memory-related config and deployment patterns

## Deployment Notes

- The API deployment mounts a PVC at `/persist` and expects a ConfigMap for `API_PORT`, `API_DB_FILEPATH`, and `CRAWLER_BASE_URL`.
- The web frontend uses its ConfigMap to point to the API backend.
- The crawler deployment runs three containers with a shared `emptyDir` volume at `/cache` and uses environment variables from its ConfigMap.
- The Gateway API stack is configured for HTTP routing via `app-gateway` and `web-httproute`.
- The `crawler` resources are namespaced under `crawler`, while the other manifests are in the default namespace unless otherwise specified.

## Quick Start

1. Apply the API resources:
   ```bash
   kubectl apply -f api-configmap.yaml
   kubectl apply -f api-pvc.yaml
   kubectl apply -f api-deployment.yaml
   kubectl apply -f api-service.yaml
   ```

2. Apply the web frontend resources:
   ```bash
   kubectl apply -f web-configmap.yaml
   kubectl apply -f web-deployment.yaml
   kubectl apply -f web-service.yaml
   ```

3. Apply the crawler resources (namespace may need to exist):
   ```bash
   kubectl create namespace crawler
   kubectl apply -f crawler-configmap.yaml
   kubectl apply -f crawler-deployment.yaml
   kubectl apply -f crawler-service.yaml
   ```

4. Apply the Gateway API resources:
   ```bash
   kubectl apply -f app-gatewayclass.yaml
   kubectl apply -f app-gateway.yaml
   kubectl apply -f web-httproute.yaml
   ```

5. Optionally enable autoscaling and resource limit examples:
   ```bash
   kubectl apply -f web-hpa.yaml
   kubectl apply -f resource_limits/testcpu-deployment.yaml
   kubectl apply -f resource_limits/testcpu-hpa.yaml
   ```

## Notes

- Ensure your cluster supports the Gateway API (`gateway.networking.k8s.io/v1`) and the specified GatewayClass controller.
- The API backend references the crawler service via DNS: `crawler-service.crawler.svc.cluster.local:80`.
- Update image tags and URLs as needed for your environment.
- This repo is intended as a manifest collection rather than a Helm chart or operator-managed deployment.
