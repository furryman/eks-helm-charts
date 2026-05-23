# Kubernetes Helm Charts

Cluster-level Helm chart sources for fuhriman.org, deployed to a k3s cluster on AWS via the [`argocd-app-of-apps`](https://github.com/furryman/argocd-app-of-apps) repo. The portfolio site lives at https://fuhriman.org; ArgoCD UI at https://argocd.fuhriman.org.

> The repo name is vestigial — it was originally created for AWS EKS; current target is k3s.

## Current Deployment

| | |
|---|---|
| Cluster | Single-node k3s on EC2 t4g.medium (4 GB ARM Graviton) |
| AMI | Packer-built; see [terraform repo](https://github.com/furryman/terraform/tree/main/packer) |
| Routing | Envoy Gateway via Gateway API (no Ingress resources) |
| TLS | cert-manager + Let's Encrypt HTTP-01 via `gatewayHTTPRoute` solver |
| DNS | ExternalDNS publishes Route53 records from HTTPRoute hostnames |
| GitOps | ArgoCD chart 9.5.15 with App-of-Apps from sibling repo |
| Infrastructure | Terraform-managed; see [terraform repository](https://github.com/furryman/terraform) |

## Charts

### `cert-manager/`

cert-manager 1.14.0 + a `letsencrypt-prod` ClusterIssuer using the **HTTP-01 `gatewayHTTPRoute` solver** that attaches to the shared `public` Gateway. `ExperimentalGatewayAPISupport` feature gate enabled (stable in 1.15+; bump when convenient).

### `envoy-gateway/`

Envoy Gateway v1.3.0 — the Gateway API implementation that replaces ingress-nginx. Bundles in the parent chart's `templates/`:

- `gatewayclass.yaml` — `envoy` GatewayClass
- `gateway.yaml` — the shared `public` Gateway in `envoy-gateway-system` with HTTP :80 and HTTPS :443 listeners; ExternalDNS target override (EIP) annotation
- `certificate.yaml` — multi-SAN cert (`fuhriman.org`, `www.fuhriman.org`, `argocd.fuhriman.org`) → Secret `fuhriman-tls` consumed by the HTTPS listener
- `argocd-server-httproute.yaml` — HTTPRoute attaching the ArgoCD UI to `argocd.fuhriman.org`

### `external-dns/`

ExternalDNS 0.15.0. Source `gateway-httproute`, provider AWS, scoped to `fuhriman.org`. Uses ambient EC2 instance-role credentials (IAM policy attached in the terraform repo's root `main.tf`). Policy `upsert-only` (safe transitional mode; never deletes records).

### `fuhriman-chart/`

The Next.js portfolio. Deployment + Service + HTTPRoute attaching to the `public` Gateway with hostnames `fuhriman.org` and `www.fuhriman.org`. Image is multi-arch (amd64+arm64), auto-published by the [`fuhriman-website`](https://github.com/furryman/fuhriman-website) repo's CI which also bumps the `tag` in `values.yaml`.

## Updating the Website

Push a change to `fuhriman-website`. Its `.github/workflows/build-deploy.yaml` builds a multi-arch image, pushes to Docker Hub, then commits a tag bump in `fuhriman-chart/values.yaml`. ArgoCD picks up the new tag on next reconciliation and rolls the deployment.

## In-cluster fuhriman.org resolution

Currently NOT rewritten at the CoreDNS layer — pod-to-fuhriman.org hairpins through the public IP. Works for the website; tricky for cert-manager HTTP-01 self-checks. A `coredns-custom/` chart with a rewrite ConfigMap is a planned follow-up.

## Chart Values

### fuhriman-chart

| Key | Description | Default |
|---|---|---|
| `image.repository` | Docker image repository | `furryman/fuhriman-website` |
| `image.tag` | Docker image tag | (auto-bumped by CI) |
| `replicaCount` | Number of replicas | `1` |
| `routing.hostnames[]` | HTTPRoute hostnames | `[fuhriman.org, www.fuhriman.org]` |
| `routing.gateway.{name,namespace}` | parentRef target | `{public, envoy-gateway-system}` |

### envoy-gateway

| Key | Description |
|---|---|
| `envoy-gateway.deployment.envoyGateway.resources` | Controller resource hints |
| `envoy-gateway.config.envoyGateway.telemetry.metrics.prometheus.disable` | `true` (no Prometheus deployed) |

### cert-manager

| Key | Description |
|---|---|
| `cert-manager.installCRDs` | `true` |
| `cert-manager.featureGates` | `ExperimentalGatewayAPISupport=true` (needed by cert-manager 1.14) |
| `letsencrypt.email` | ACME account email |
| `letsencrypt.server` | ACME directory URL (production) |

### external-dns

| Key | Description |
|---|---|
| `external-dns.provider.name` | `aws` |
| `external-dns.policy` | `upsert-only` |
| `external-dns.sources` | `[gateway-httproute]` |
| `external-dns.domainFilters` | `[fuhriman.org]` |
