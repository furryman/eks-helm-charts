# EKS Helm Charts

This repository contains Helm charts for the fuhriman.org Kubernetes infrastructure.

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
