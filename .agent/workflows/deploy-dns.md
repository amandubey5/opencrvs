---
description: Deploy OpenCRVS with DNS on Kubernetes (Traefik ACME + Let's Encrypt)
---

# Deploy OpenCRVS with DNS on Kubernetes

## Before you start — set your domain and email

In `traefik/values.yaml`: set `email: your-email@example.com`
In `opencrvs-services/values.yaml` and `dependencies/values.yaml`: set `hostname: your-domain.com`

## 1. Install Traefik

```bash
helm upgrade --install traefik oci://ghcr.io/traefik/helm/traefik \
  --namespace traefik \
  --create-namespace \
  -f infrastructure/examples/dns-deployment/traefik/values.yaml \
  --wait
```

// turbo
Get the LoadBalancer external IP:
```bash
kubectl get svc -n traefik
```

## 2. Set DNS A records (Manual)

In your DNS provider, create:
- `A  your-domain.com    →  <EXTERNAL-IP>`
- `A  *.your-domain.com  →  <EXTERNAL-IP>`

Wait for DNS propagation (~1–2 minutes) before continuing.

## 3. Install OpenCRVS Dependencies

```bash
helm upgrade --install opencrvs-deps oci://ghcr.io/opencrvs/opencrvs-dependencies-chart \
  --namespace "opencrvs-deps-dev" \
  --create-namespace \
  --atomic \
  --timeout 10m \
  -f infrastructure/examples/dns-deployment/dependencies/values.yaml
```

## 4. Install OpenCRVS Services

```bash
helm upgrade --install opencrvs oci://ghcr.io/opencrvs/opencrvs-services \
  --timeout 1h \
  --namespace "opencrvs-dev" \
  --create-namespace \
  --atomic \
  -f infrastructure/examples/dns-deployment/opencrvs-services/values.yaml
```

// turbo
## 5. Verify

```bash
kubectl get pods -n opencrvs-dev && kubectl get ingressroute -n opencrvs-dev
```
