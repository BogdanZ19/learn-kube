# SynergyChat - Kubernetes Deployment Configuration

A microservices-based chat platform with web scraping capabilities deployed on Kubernetes, featuring auto-scaling, persistent storage, and resource management patterns.

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Project Structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Key Features](#key-features)
- [File Organization](#file-organization)
- [Resource Testing](#resource-testing)

## Overview

**SynergyChat** is a distributed application running on Kubernetes with three primary services:

- **Web**: Frontend UI with horizontal auto-scaling capabilities
- **API**: Backend service with persistent data storage
- **Crawler**: Background worker service for web scraping and keyword analysis

The services communicate through the Kubernetes Gateway API, providing clean internal DNS-based routing without exposing services directly outside the cluster.

## Architecture

### Service Topology

```
┌─────────────────────────────────────────────────────────┐
│ Kubernetes Cluster                                      │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Gateway (app-gateway.yaml)                             │
│  └─> HTTP Route: web-httproute.yaml & api-httproute     │
│                                                         │
│  ┌──────────────────┐      ┌──────────────────┐         │
│  │ Web Service      │      │ API Service      │         │
│  │ (3 replicas)     │      │ (1 replica)      │         │
│  │ Auto-scaling     │◄────►│ PVC: 1Gi         │         │
│  │ 1-4 replicas     │      │ JSON database    │         │
│  │ (50% CPU)        │      │                  │         │
│  └──────────────────┘      └──────────────────┘         │
│         ▲                                               │
│         │                                               │
│         ▼                                               │
│  ┌─────────────────────────────────────────────────┐    │
│  │ Crawler Service (crawler namespace)             │    │
│  │ 3x container instances (ports 8080-8082)        │    │
│  │ Parallel keyword analysis & web scraping        │    │
│  └─────────────────────────────────────────────────┘    │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Namespaces

- **default**: Web and API services
- **crawler**: Isolated crawler worker namespace

## Project Structure

### Core Services

| File | Purpose |
|------|---------|
| `web-deployment.yaml` | Frontend UI deployment with 3 initial replicas |
| `web-service.yaml` | Internal Service for web frontend |
| `web-httproute.yaml` | Gateway HTTP routing for web UI |
| `web-configmap.yaml` | Web service configuration (environment variables, settings) |
| `web-hpa.yaml` | Horizontal Pod Autoscaler (1-4 replicas based on CPU) |
| `api-deployment.yaml` | Backend API deployment |
| `api-service.yaml` | Internal Service for API backend |
| `api-httproute.yaml` | Gateway HTTP routing for API |
| `api-configmap.yaml` | API configuration |
| `api-pvc.yaml` | PersistentVolumeClaim (1Gi) for JSON database storage |
| `crawler-deployment.yaml` | Crawler worker service with 3 parallel containers |
| `crawler-service.yaml` | Internal Service for crawler (if needed for inter-pod communication) |
| `crawler-configmap.yaml` | Crawler keywords and configuration |

### Gateway Configuration

| File | Purpose |
|------|---------|
| `app-gatewayclass.yaml` | GatewayClass definition for cluster routing |
| `app-gateway.yaml` | Gateway resource managing HTTP routes and load balancing |

### Resource Testing

| File | Purpose |
|------|---------|
| `resource_limits/testcpu-deployment.yaml` | Test deployment with strict CPU limits (50m) for eviction testing |
| `resource_limits/testcpu-hpa.yaml` | HPA for CPU test deployment |
| `resource_limits/testram-deployment.yaml` | Test deployment with memory limits (256Mi) for OOMKilled scenarios |
| `resource_limits/testram-configmap.yaml` | Configuration for memory test deployment |

## Prerequisites

- Kubernetes cluster (v1.21+) with Gateway API support enabled
- `kubectl` installed and configured to access your cluster
- A StorageClass available for PersistentVolumeClaim (check: `kubectl get storageclass`)
- Gateway API resources installed (if not built-in to your cluster)

## Quick Start

### 1. Verify Gateway API Support

```bash
kubectl api-resources | grep gateway
```

If not available, install the Gateway API CRDs:
```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.0.0/standard-install.yaml
```

### 2. Deploy GatewayClass and Gateway

```bash
kubectl apply -f app-gatewayclass.yaml
kubectl apply -f app-gateway.yaml
```

### 3. Create Crawler Namespace

```bash
kubectl create namespace crawler
```

### 4. Deploy Core Services

Apply configurations in this order:

```bash
# API with storage
kubectl apply -f api-pvc.yaml
kubectl apply -f api-configmap.yaml
kubectl apply -f api-deployment.yaml
kubectl apply -f api-service.yaml
kubectl apply -f api-httproute.yaml

# Web frontend
kubectl apply -f web-configmap.yaml
kubectl apply -f web-deployment.yaml
kubectl apply -f web-service.yaml
kubectl apply -f web-httproute.yaml
kubectl apply -f web-hpa.yaml

# Crawler workers
kubectl apply -f crawler-configmap.yaml -n crawler
kubectl apply -f crawler-deployment.yaml -n crawler
kubectl apply -f crawler-service.yaml -n crawler
```

### 5. Verify Deployment

```bash
# Check all pods
kubectl get pods -A

# Check services
kubectl get svc

# Check Gateway and routes
kubectl get gateway
kubectl get httproute -A

# Check PVC is bound
kubectl get pvc
```

## Key Features

### Horizontal Auto-Scaling
The Web service automatically scales between 1-4 replicas based on 50% CPU utilization. Monitor scaling with:
```bash
kubectl get hpa web-hpa -w
```

### Persistent Storage
The API service stores data in a 1Gi PersistentVolume. Verify with:
```bash
kubectl get pvc api-pvc
kubectl describe pvc api-pvc
```

### Multi-Container Crawler
The crawler pod runs 3 container instances in parallel on different ports (8080, 8081, 8082), each analyzing keywords: `love`, `hate`, `joy`, `sadness`, `anger`, `disgust`, `fear`, `surprise`.

### Service Discovery
Services communicate via internal DNS:
- Web UI: `http://synchat.internal`
- API backend: `http://synchatapi.internal`

### Namespace Isolation
Crawler service runs in `crawler` namespace for resource isolation and separate configuration management.

## File Organization

The project is organized by component and layer:

```
kube/
├── README.md (this file)
├── app-gateway*.yaml          # Gateway API resources
├── web-*.yaml                 # Web service (frontend)
├── api-*.yaml                 # API service (backend)
├── crawler-*.yaml             # Crawler service (workers)
└── resource_limits/
    ├── testcpu-*.yaml         # CPU resource limit testing
    └── testram-*.yaml         # Memory resource limit testing
```

**Naming Convention**: `<service>-<resource-type>.yaml`
- `<service>`: web, api, crawler, app
- `<resource-type>`: deployment, service, configmap, httproute, pvc, hpa, gatewayclass, gateway

## Resource Testing

The `resource_limits/` directory contains configurations for testing Kubernetes resource management:

### CPU Testing (`testcpu-deployment.yaml`)
- Deployments with CPU limit of 50m (50 millicores)
- Tests pod eviction and throttling under CPU constraints
- Uses HPA to observe scaling behavior under limited resources

### Memory Testing (`testram-deployment.yaml`)
- Deployments with memory limit of 256Mi
- Tests OOMKilled (Out of Memory) scenarios
- Validates pod crash loops and recovery mechanisms

Use these deployments to validate resource quotas, limits, and node capacity planning:

```bash
# Deploy CPU test
kubectl apply -f resource_limits/testcpu-deployment.yaml
kubectl apply -f resource_limits/testcpu-hpa.yaml

# Deploy memory test
kubectl apply -f resource_limits/testram-configmap.yaml
kubectl apply -f resource_limits/testram-deployment.yaml

# Monitor resource usage
kubectl top pods
kubectl top nodes
```

---


