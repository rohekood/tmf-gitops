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
kubectl apply -f clusters/dev/release/infra/ingress-nginx-repository.yaml
kubectl apply -f clusters/dev/release/infra/ingress-nginx.yaml
kubectl apply -f clusters/dev/release/infra/metallb-ingress-nginx-pool.yaml
kubectl apply -f clusters/dev/release/apps/tmf-platform.yaml
kubectl apply -f clusters/dev/release/apps/tmf-platform-tls.yaml
kubectl apply -f clusters/dev/release/apps/tmf-platform-ingress.yaml
kubectl apply -R -f clusters/dev/images
kubectl apply -R -f clusters/dev/automation
```

## ingress-nginx with MetalLB

- ingress-nginx is deployed via Helm chart `ingress-nginx` in namespace `ingress-nginx`.
- ingress-nginx controller service is `LoadBalancer` and receives an external IP from MetalLB.
- A dedicated MetalLB range is reserved for ingress-nginx in `clusters/dev/release/infra/metallb-ingress-nginx-pool.yaml` (`192.168.10.40-192.168.10.45`).
- The existing `first-pool` is split to avoid overlap (`10-39` and `46-50`), and `40-45` is reserved for ingress-nginx.
- tmf-platform keeps `demo-ui` and `bff` as `ClusterIP` and exposes routes via `clusters/dev/release/apps/tmf-platform-ingress.yaml`.
- tmf-platform ingress routes:
	- `/api` to bff service
	- `/` to demo-ui service

After applying manifests, verify:

```bash
kubectl get svc -n emissary-system
kubectl get svc -n ingress-nginx
kubectl get ingress -n tmf
kubectl get helmreleases -A
```

## Runtime secrets (required)

Each service has its own Kubernetes Secret. Secrets are **never committed to git** — apply them directly with kubectl.

Each service reads `RABBITMQ_URL` from its own secret, allowing per-service RabbitMQ credentials.
Before creating secrets, run `tmf/scripts/rabbitmq-setup-prod.sh` to provision broker users.

> **CloudAMQP note**: The vhost is fixed to your account name (e.g. `evlrxeci`). Pass `RABBIT_VHOST=evlrxeci` when running the setup script. Vhost creation will be skipped if it already exists.

```bash
# party-management
kubectl create secret generic party-management-secrets -n tmf \
  --from-literal=RABBITMQ_URL='amqps://tmf_party:<password>@dog.lmq.cloudamqp.com/evlrxeci' \
  --from-literal=PARTY_DB_URL='postgres://user:pass@host:5432/tmf_party_db?sslmode=require'

# customer-management
kubectl create secret generic customer-management-secrets -n tmf \
  --from-literal=RABBITMQ_URL='amqps://tmf_customer:<password>@dog.lmq.cloudamqp.com/evlrxeci' \
  --from-literal=CUSTOMER_DB_URL='postgres://user:pass@host:5432/tmf_customer_db?sslmode=require'

# product-catalog-management
kubectl create secret generic product-catalog-management-secrets -n tmf \
  --from-literal=RABBITMQ_URL='amqps://tmf_catalog:<password>@dog.lmq.cloudamqp.com/evlrxeci' \
  --from-literal=CATALOG_DB_URL='postgres://user:pass@host:5432/tmf_catalog_db?sslmode=require'

# qualification
kubectl create secret generic qualification-secrets -n tmf \
  --from-literal=RABBITMQ_URL='amqps://tmf_qualification:<password>@dog.lmq.cloudamqp.com/evlrxeci' \
  --from-literal=QUALIFICATION_DB_URL='postgres://user:pass@host:5432/tmf_qualification_db?sslmode=require' \
  --from-literal=REDIS_URL='redis://...' \
  --from-literal=REDIS_PASSWORD='<redis-password>'

# shopping-cart
kubectl create secret generic shopping-cart-secrets -n tmf \
  --from-literal=RABBITMQ_URL='amqps://tmf_cart:<password>@dog.lmq.cloudamqp.com/evlrxeci' \
  --from-literal=CART_DB_URL='postgres://user:pass@host:5432/tmf_cart_db?sslmode=require'

# pocv
kubectl create secret generic pocv-secrets -n tmf \
  --from-literal=RABBITMQ_URL='amqps://tmf_pocv:<password>@dog.lmq.cloudamqp.com/evlrxeci' \
  --from-literal=POCV_DB_URL='postgres://user:pass@host:5432/tmf_pocv_db?sslmode=require'

# bff
kubectl create secret generic bff-secrets -n tmf \
  --from-literal=RABBITMQ_URL='amqps://tmf_bff:<password>@dog.lmq.cloudamqp.com/evlrxeci' \
  --from-literal=AUTH0_DOMAIN='your-tenant.eu.auth0.com' \
  --from-literal=AUTH0_AUDIENCE='https://api.your-domain.com'

# demo-ui (no RabbitMQ, only Auth0)
kubectl create secret generic demo-ui-secrets -n tmf \
  --from-literal=AUTH0_DOMAIN='your-tenant.eu.auth0.com' \
  --from-literal=AUTH0_CLIENT_ID='your-client-id' \
  --from-literal=AUTH0_AUDIENCE='https://api.your-domain.com'
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
