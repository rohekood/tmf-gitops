# tmf-gitops

Flux release manifests for deploying tmf-ai-demo into Kubernetes.

## Repository layout

- clusters/dev/release/release.yaml: GitRepository source for tmf-ai-demo charts.
- clusters/dev/release/apps/tmf-platform.yaml: single umbrella HelmRelease for all TMF apps.
- clusters/dev/images/*/imagerepository.yaml: image scan sources.
- clusters/dev/images/*/imagepolicy.yaml: image selection policies.
- clusters/dev/automation/gitrepository.yaml: GitRepository used by image automation.
- clusters/dev/automation/image-update.yaml: ImageUpdateAutomation for tag updates.

- tmf-ai-demo source: ssh://git@github.com/rohekood/tmf-ai-demo
- umbrella chart path: ./releases/charts/tmf-platform

## Prerequisites

- kubectl
- flux
- Access to this repo and cluster credentials
- Flux controllers already installed in cluster

## Deploy tmf-ai-demo

Apply manifests in this order:

```bash
kubectl apply -f clusters/dev/release/release.yaml
kubectl apply -f clusters/dev/release/infra/emissary-repository.yaml
kubectl apply -f clusters/dev/release/infra/emissary.yaml
kubectl apply -f clusters/dev/release/apps/tmf-platform.yaml
kubectl apply -R -f clusters/dev/images
kubectl apply -R -f clusters/dev/automation
```

## Emissary with MetalLB

- Emissary is deployed as a `LoadBalancer` service and receives an external IP from MetalLB.
- A dedicated MetalLB range is reserved for Emissary in `clusters/dev/release/infra/metallb-emissary-pool.yaml` (`192.168.10.40-192.168.10.45`).
- tmf-platform now keeps `demo-ui` as `ClusterIP` and exposes app traffic through Emissary mappings.
- tmf-platform Emissary mappings route:
	- `/api/` to bff service
	- `/` to demo-ui service

After applying manifests, verify:

```bash
kubectl get svc -n emissary-system
kubectl get mappings.getambassador.io -n tmf
kubectl get helmreleases -A
```

## Runtime secrets (required)

Credentialed connection strings must be stored in Kubernetes Secrets, not inline Helm values.

The umbrella HelmRelease uses `env.secretName: tmf-secrets` for backend services, so create this secret in namespace `tmf` with these keys:

- `RABBITMQ_URL`
- `CUSTOMER_DB_URL`
- `PARTY_DB_URL`
- `CATALOG_DB_URL`
- `CART_DB_URL`
- `POCV_DB_URL`
- `QUALIFICATION_DB_URL`
- `REDIS_PASSWORD` (only if your Redis setup requires auth)

Example:

```bash
kubectl create secret generic tmf-secrets \
	-n tmf \
	--from-literal=RABBITMQ_URL='amqps://user:pass@host/vhost' \
	--from-literal=CUSTOMER_DB_URL='postgres://user:pass@host:5432/db?sslmode=require' \
	--from-literal=PARTY_DB_URL='postgres://user:pass@host:5432/db?sslmode=require' \
	--from-literal=CATALOG_DB_URL='postgres://user:pass@host:5432/db?sslmode=require' \
	--from-literal=CART_DB_URL='postgres://user:pass@host:5432/db?sslmode=require' \
	--from-literal=POCV_DB_URL='postgres://user:pass@host:5432/db?sslmode=require' \
	--from-literal=QUALIFICATION_DB_URL='postgres://user:pass@host:5432/db?sslmode=require'
```

## Image automation

This repo now includes:

- `ImageRepository` and `ImagePolicy` manifests for each tmf-ai-demo service
- a dedicated `GitRepository` named `tmf-gitops`
- an `ImageUpdateAutomation` named `tmf-gitops-update`

Both GitRepository objects currently use the secret name enercon-release-auth.
That secret must have SSH access to:

- git@github.com/rohekood/tmf-ai-demo
- git@github.com/rohekood/tmf-gitops

Example:

```bash
flux create secret git enercon-release-auth \
	--url=ssh://git@github.com/rohekood/tmf-gitops \
	--private-key-file=/path/to/id_ed25519 \
	-n flux-system
```

Without this secret, source fetch and image automation push will not become ready.

Then reconcile:

```bash
flux reconcile source git tmf-ai-demo -n flux-system
flux reconcile helmrelease tmf-platform -n tmf
flux reconcile image update tmf-gitops-update -n flux-system
```

## Verify

```bash
flux get sources git -A
flux get kustomizations -A
flux get helmreleases -A
flux get image repository -A
flux get image policy -A
flux get image update -A
```
