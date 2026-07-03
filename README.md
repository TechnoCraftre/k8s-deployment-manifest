# k8s-deployment-manifest

Production-grade Kubernetes manifests for deploying a containerized web application with rolling updates, auto-scaling, health checks, and AWS ALB ingress.

## What It Does

- Deploys 3 replicas of an Nginx-based application with zero-downtime rolling updates
- Exposes the app via ClusterIP service for internal communication
- Routes external traffic via AWS ALB Ingress Controller with HTTPS termination
- Auto-scales pods based on CPU (70%) and memory (80%) utilization
- Configures liveness and readiness probes for self-healing
- Uses pod anti-affinity to distribute pods across nodes for high availability

## Tech Stack

- **Orchestration**: Kubernetes (EKS/AKS/GKE compatible)
- **Ingress**: AWS ALB Ingress Controller
- **Auto-scaling**: HPA v2
- **Container**: Nginx Alpine
- **Networking**: ClusterIP + ALB (Layer 7)

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                        Internet                             │
└─────────────────────┬───────────────────────────────────────┘
                      │ HTTPS (443)
                      ▼
┌─────────────────────────────────────────────────────────────┐
│              AWS ALB (Application Load Balancer)            │
│              • SSL termination                              │
│              • Health checks: /health                       │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│              Ingress: portfolio-ingress                     │
│              • Host: portfolio.example.com                  │
│              • Path: /                                      │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│              Service: portfolio-service                     │
│              Type: ClusterIP                                │
│              Port: 80 → 8080                                │
└─────────────────────┬───────────────────────────────────────┘
                      │
        ┌─────────────┼─────────────┐
        │             │             │
        ▼             ▼             ▼
┌───────────┐ ┌───────────┐ ┌───────────┐
│   Pod 1   │ │   Pod 2   │ │   Pod 3   │
│  (Node A) │ │  (Node B) │ │  (Node C) │
│  128Mi/   │ │  128Mi/   │ │  128Mi/   │
│  100m CPU │ │  100m CPU │ │  100m CPU │
│           │ │           │ │           │
│ Liveness  │ │ Liveness  │ │ Liveness  │
│ Readiness │ │ Readiness │ │ Readiness │
└───────────┘ └───────────┘ └───────────┘
        │             │             │
        └─────────────┴─────────────┘
                      │
              ┌───────▼───────┐
              │     HPA       │
              │ 3-10 replicas │
              │ CPU > 70%     │
              │ Mem > 80%     │
              └───────────────┘
```

## How to Run It

### Prerequisites

- kubectl configured with cluster access
- AWS ALB Ingress Controller installed (for EKS)
- ExternalDNS or manual Route53 configuration

### Deployment Steps

```bash
# Apply all manifests in order
cd k8s-deployment-manifest

# Create ConfigMap first (required by Deployment)
kubectl apply -f configmap.yaml

# Deploy application
kubectl apply -f deployment.yaml

# Expose via Service
kubectl apply -f service.yaml

# Configure Ingress (requires ALB Controller + ACM certificate)
kubectl apply -f ingress.yaml

# Enable auto-scaling
kubectl apply -f hpa.yaml

# Verify rollout
kubectl get pods -l app=portfolio-app -w
kubectl get svc portfolio-service
kubectl get ingress portfolio-ingress
```

### Blue-Green Deployment Strategy

```bash
# Current: v1 (green)
# Update image to v2 (blue)
kubectl set image deployment/portfolio-app app=nginx:1.25 --record

# Monitor rollout
kubectl rollout status deployment/portfolio-app

# If issues, rollback immediately
kubectl rollout undo deployment/portfolio-app
```

### Scaling Tests

```bash
# Generate load to trigger HPA
kubectl run load-generator --image=busybox -- /bin/sh -c "while true; do wget -q -O- http://portfolio-service; done"

# Watch HPA scale up
kubectl get hpa portfolio-hpa -w
```

## Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| `maxUnavailable: 0` | Zero-downtime deployments |
| `podAntiAffinity` | High availability across nodes |
| `ClusterIP` + ALB | Security: pods not directly exposed |
| `stabilizationWindowSeconds: 300` | Prevent flapping during scale-down |
| Resource limits | Prevent noisy neighbor issues |

## Files

| File | Purpose |
|------|---------|
| `deployment.yaml` | Main app deployment with probes |
| `service.yaml` | Internal service discovery |
| `ingress.yaml` | External routing via ALB |
| `configmap.yaml` | Nginx configuration |
| `hpa.yaml` | Horizontal pod autoscaling |

