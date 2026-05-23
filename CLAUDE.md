# CLAUDE.md

Guidance for Claude Code working in this repository.

This repo holds the **chart sources** deployed to the fuhriman.org k3s cluster. ArgoCD reconciles each chart in here continuously, driven by the [`argocd-app-of-apps`](https://github.com/furryman/argocd-app-of-apps) parent chart on the same cluster. The AWS-side infrastructure (VPC, EC2, Route53, etc.) lives in [`furryman/terraform`](https://github.com/furryman/terraform); the actual website code lives in [`furryman/fuhriman-website`](https://github.com/furryman/fuhriman-website).

> **Name is vestigial.** This repo was originally created for AWS EKS. Current deploy target is **single-node k3s** on a `t4g.medium` (ARM Graviton). Don't add EKS-only constructs.

## Commands

There is no build step or test suite in this repo — everything is YAML and Helm templates. The relevant commands are mostly for verification:

```bash
# Lint every chart
for d in cert-manager envoy-gateway external-dns fuhriman-chart; do
  helm lint "$d"
done

# Render a chart locally for diff review
helm template fuhriman-chart fuhriman-chart/ | less

# Update a dependency lock for a wrapper chart
(cd cert-manager && helm dependency update)

# Diff-apply through ArgoCD (preferred — see the App-of-Apps repo for argo CLI usage)
```

You almost never `helm install` from a workstation; ArgoCD is the deployer. Edits land on `main`, ArgoCD picks them up on next reconcile (or via a manual `argocd app sync <app>`).

## Repository layout

```text
eks-helm-charts/
├── cert-manager/         # cert-manager 1.14.0 + ClusterIssuer
├── envoy-gateway/        # Envoy Gateway v1.3.0 + shared Gateway, Certificate, ArgoCD HTTPRoute
├── external-dns/         # ExternalDNS 0.15.0 — Route53 from HTTPRoute hostnames
└── fuhriman-chart/       # The portfolio Deployment + Service + HTTPRoute
```

Each chart follows the same pattern:

- `Chart.yaml` — `version: 1.0.0`, `appVersion` pinned to the upstream release. `dependencies:` pulls the real chart from upstream and aliases it; this repo's chart is just a thin wrapper.
- `values.yaml` — overrides for the upstream chart, plus a top-level block of values consumed by this repo's own templates.
- `templates/` — k8s manifests this repo *owns* (not what the upstream chart renders). Used for cluster-wide singletons that don't belong inside any one component's chart.

## Chart-by-chart

### `cert-manager/` (sync wave -2)

Wraps `jetstack/cert-manager` 1.14.0. The only template owned here is:

- `templates/cluster-issuer.yaml` — `letsencrypt-prod` ClusterIssuer using the **HTTP-01 `gatewayHTTPRoute` solver** that attaches to the shared `public` Gateway in `envoy-gateway-system`.

**`featureGates: "ExperimentalGatewayAPISupport=true"`** is set in `values.yaml`. cert-manager 1.14 requires this feature gate for Gateway API support; 1.15+ makes it stable. When you bump the chart, drop the gate.

### `envoy-gateway/` (sync wave -1)

Wraps `oci://docker.io/envoyproxy/gateway-helm` v1.3.0. The most opinionated chart in the repo — owns four cluster-singleton resources used by everything else:

- `templates/gatewayclass.yaml` — `envoy` GatewayClass.
- `templates/gateway.yaml` — the shared `public` Gateway in `envoy-gateway-system`, listeners `http:80` + `https:443`. Carries two annotations consumed by ExternalDNS:
  - `external-dns.alpha.kubernetes.io/target: "52.37.95.130"` — the EIP. Override needed because klipper-lb (k3s default LoadBalancer) advertises the node's private IP in the Service's LB status, which would otherwise be what ExternalDNS publishes.
  - `external-dns.alpha.kubernetes.io/hostname: "fuhriman.org,www.fuhriman.org"` — the apex hostnames published against the Gateway itself (separate from any HTTPRoute hostnames).
- `templates/certificate.yaml` — multi-SAN `Certificate` named `fuhriman-tls` covering `fuhriman.org`, `www.fuhriman.org`, `argocd.fuhriman.org`. Issued by `letsencrypt-prod` into Secret `fuhriman-tls` in `envoy-gateway-system`, which the Gateway's HTTPS listener references.
- `templates/argocd-server-httproute.yaml` — HTTPRoute attaching ArgoCD's UI Service to `argocd.fuhriman.org`. **Lives in this chart on purpose**: putting it in the `argocd-app-of-apps` chart would race the Gateway API CRDs on a fresh cluster (CRDs come from the envoy-gateway chart).

`values.yaml` keeps the controller's Prometheus emitter disabled (no Prometheus in this cluster), and trims resource requests/limits.

### `external-dns/` (sync wave 0)

Wraps `external-dns-sigs/external-dns` 1.15.0. Configuration is in `values.yaml`:

- `provider.name: aws`
- `sources: [gateway-httproute]` — Gateway API native; do not add `ingress` (no Ingress resources in this cluster).
- `domainFilters: [fuhriman.org]`
- `policy: upsert-only` — safe transitional mode; never deletes Route53 records. Once steady-state is verified, this can be switched to `sync` for full reconciliation, but it's not load-bearing — flip with care.
- `registry: txt` + `txtPrefix: _externaldns.` + `txtOwnerId: fuhriman-portfolio` — TXT-registry semantics so multiple ExternalDNS instances couldn't fight over the same records (and as a safety rail against orphaned manual records).

ExternalDNS uses **ambient EC2 instance-role credentials**. The IAM policy granting Route53 record management on the public zone is attached to the k3s node's instance role in the terraform repo's root `main.tf`. There is no IRSA, no serviceAccount annotation, no AWS_ACCESS_KEY in values. If you add another DNS-managing controller, attach to the same instance role rather than duplicating credentials.

### `fuhriman-chart/` (sync wave 0)

The actual portfolio website. Templates owned here:

- `_helpers.tpl` — standard helm name/labels helpers.
- `deployment.yaml`, `service.yaml` — Deployment + ClusterIP Service.
- `httproute.yaml` — HTTPRoute attaching to the shared `public` Gateway (via `parentRefs.sectionName: https`) with hostnames from `routing.hostnames` in values.

`values.yaml`:

- `image.repository: furryman/fuhriman-website`
- `image.tag` — auto-bumped by `fuhriman-website`'s CI after a multi-arch build + Docker Hub push. **Don't hand-edit the tag** unless you're rolling back; the CI commit will revert it.
- `routing.gateway: {name: public, namespace: envoy-gateway-system}` — `parentRef` target.
- `routing.hostnames: [fuhriman.org, www.fuhriman.org]`.

## Architectural conventions

- **Gateway API only.** There are zero `Ingress` resources in this cluster. New hostnames are added by:
  1. Adding the SAN to `envoy-gateway/templates/certificate.yaml`.
  2. Creating a new HTTPRoute that attaches to `parentRefs.public` in `envoy-gateway-system`.

  Don't create a second Gateway unless you have a reason that justifies a second LB (you probably don't on this cluster).
- **Sync-wave ownership.** CRDs and cluster-wide infra precede workloads. `cert-manager` (-2) → `envoy-gateway` (-1) → `external-dns` (0), `fuhriman-website` (0). If a chart starts to depend on something in a higher-wave chart, restructure the chart, not the waves.
- **Wrapper chart pattern.** `Chart.yaml` `dependencies:` pulls upstream. Top-level keys in `values.yaml` (e.g. `letsencrypt:`, `routing:`) belong to *this repo*. Keys nested under the alias (e.g. `cert-manager:`, `external-dns:`) get passed to the upstream chart. Keep the boundary clean.
- **Images must be multi-arch.** The cluster runs on ARM Graviton; `fuhriman-website`'s CI builds `linux/amd64,linux/arm64`. If you add a new workload, verify the upstream image supports `linux/arm64` (most do — confirm with `docker manifest inspect`).
- **No Prometheus.** Charts disable Prometheus metrics emitters in values (envoy-gateway, external-dns). Don't paste in default Prometheus annotations.

## Sibling repos

- **`furryman/terraform`** — VPC, EC2 + Packer AMI, Route53 zone, IAM (including the Route53 policy used by ExternalDNS), S3 state backend. Look here if a chart needs to reference cloud state.
- **`furryman/argocd-app-of-apps`** — Helm chart declaring the four `Application` CRs that point at the four directories in this repo.
- **`furryman/fuhriman-website`** — Next.js source for the portfolio. Its `build-deploy` workflow auto-bumps `fuhriman-chart/values.yaml` after a successful image push.

## When working in this repo

**Defaults**:

- The deployed cluster is the source of truth. Run `kubectl get gateway,httproute,certificate -A` over the SSM tunnel to see real state; don't infer it from this repo alone.
- Most edits are values-only. Reach for `values.yaml` before adding new templates.
- When you bump an `appVersion`/dependency `version`, also bump the wrapper chart's `version` in `Chart.yaml` so ArgoCD reliably detects a change.

**Gotchas**:

- The `external-dns` annotation on the shared Gateway is the **only** place the EIP is hardcoded in this repo. If the EIP ever changes (it shouldn't — it's Elastic), update `envoy-gateway/templates/gateway.yaml` here. The terraform repo emits the EIP value via output if you need to verify.
- cert-manager 1.14's Gateway API support is behind `ExperimentalGatewayAPISupport=true`. If you upgrade past 1.15, remove that flag — it'll become an error.
- The `argocd-server` HTTPRoute lives in `envoy-gateway/templates/` (not in the App-of-Apps repo) deliberately. Don't move it.
- `external-dns` policy is `upsert-only`. It will never delete records. If you remove a hostname and want the corresponding A record cleaned up, you must do that out of band.
- The website image tag is rewritten by CI. Local commits that touch `fuhriman-chart/values.yaml`'s `image.tag` will be overwritten on the next website CI run. Keep manual edits to other keys.
- This cluster does not have CoreDNS rewrites for `fuhriman.org` yet. Pod-internal lookups hairpin through the public EIP. Acceptable today, planned to land as a `coredns-custom/` chart.
