# tmf-gitops

Flux release manifests for deploying tmf-ai-demo into Kubernetes.

## Repository layout

- `clusters/dev/release/release.yaml`: tmf-ai-demo source and kustomization.
- `clusters/dev/kustomization.yaml`: root release-only kustomization.

The release manifest points Flux to:

- `https://github.com/rohekood/tmf-ai-demo`
- `./releases/flux` path in that repository.

## Prerequisites

- `kubectl`
- `flux`
- Access to this repo and cluster credentials

## Deploy tmf-ai-demo

This repository assumes Flux controllers are already installed in the cluster.

## Image automation

This repo now includes:

- `ImageRepository` and `ImagePolicy` manifests for each tmf-ai-demo service
- a dedicated `GitRepository` named `tmf-gitops`
- an `ImageUpdateAutomation` named `tmf-gitops-update`

The automation uses SSH push access to this repository and requires a Flux git secret named `tmf-gitops-auth` in the `flux-system` namespace.

Example:

```bash
flux create secret git tmf-gitops-auth \
	--url=ssh://git@github.com/rohekood/tmf-gitops \
	--private-key-file=/path/to/id_ed25519 \
	-n flux-system
```

Without that secret, the `tmf-gitops` GitRepository and `tmf-gitops-update` automation will not become ready.

Create or update release objects:

```bash
kubectl apply -k clusters/dev
```

Or apply only the release object:

```bash
kubectl apply -f clusters/dev/release/release.yaml
```

Then reconcile:

```bash
flux reconcile source git tmf-ai-demo -n flux-system
flux reconcile kustomization tmf-ai-demo -n flux-system
```

## Verify

```bash
flux get sources git -A
flux get kustomizations -A
flux get helmreleases -A
```
