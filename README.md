# Kubernetes Helm Charts

This repository contains Helm charts for the fuhriman.org Kubernetes infrastructure, currently deployed to a k3s cluster on AWS EC2.

> **Note:** This repository was originally created for AWS EKS but now serves a k3s deployment. The charts are platform-agnostic and work on any Kubernetes cluster.

## Current Deployment

These charts are deployed to a lightweight k3s Kubernetes cluster running on a single AWS EC2 t3.micro instance. ArgoCD manages continuous deployment using the app-of-apps pattern defined in [argocd-app-of-apps](https://github.com/furryman/argocd-app-of-apps).

**Infrastructure:**
- **Cluster:** Single-node k3s on Amazon Linux 2023
- **Instance:** t3.micro (free tier eligible)
- **GitOps:** ArgoCD with auto-sync, prune, and self-heal
- **Deployment:** Terraform-managed (see [terraform repository](https://github.com/furryman/terraform))

## Charts

### cert-manager

TLS certificate management using Let's Encrypt.

```bash
cd cert-manager
helm dependency update
helm install cert-manager . -n cert-manager --create-namespace
```

### ingress-nginx

NGINX Ingress Controller for handling incoming traffic.

```bash
cd ingress-nginx
helm dependency update
helm install ingress-nginx . -n ingress-nginx --create-namespace
```

### fuhriman-chart

The portfolio website application.

```bash
cd fuhriman-chart
helm install fuhriman . -n default
```

## ArgoCD Integration

These charts are deployed automatically via ArgoCD using the app-of-apps pattern. The ArgoCD Application definitions are in the [argocd-app-of-apps](https://github.com/furryman/argocd-app-of-apps) repository.

## Updating the Website

The `fuhriman-chart/values.yaml` file contains the image tag. This is automatically updated by the CI/CD pipeline in the [fuhriman-website](https://github.com/furryman/fuhriman-website) repository when changes are pushed to main.

## Chart Values

### fuhriman-chart

| Key | Description | Default |
|-----|-------------|---------|
| `image.repository` | Docker image repository | `furryman/fuhriman-website` |
| `image.tag` | Docker image tag | `latest` |
| `replicaCount` | Number of replicas | `2` |
| `ingress.hosts[0].host` | Hostname | `fuhriman.org` |
