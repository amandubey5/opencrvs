# OpenCRVS on Kubernetes — Bare-Metal Single Node

Complete end-to-end deployment guide. All values are pre-filled. No edits required.

**Domain:** `opencrvs.com` | **Country:** Farajaland | **Version:** OpenCRVS v1.9.9

---

## Namespaces

| Namespace | Created By | Contains |
|-----------|-----------|----------|
| `local-path-storage` | Step 3 (`kubectl apply`) | Local path storage provisioner |
| `traefik` | Step 4 (`helm install`) | Traefik ingress controller |
| `cert-manager` | Step 5 (`helm install`) | cert-manager controller |
| `opencrvs-dev` | Step 6 (`kubectl create`) | Pre-created for TLS certificate |
| `opencrvs-deps-dev` | Step 8 (`helm install`) | MongoDB, PostgreSQL, Elasticsearch, Redis, MinIO, InfluxDB |
| `opencrvs-dev` | Step 9 (`helm install`) | All OpenCRVS application services |

---

## How TLS works in this setup

```
cert-manager (namespace: cert-manager)
    │  HTTP-01 challenge (port 80 via Traefik Ingress)
    │  contacts Let's Encrypt → certificate issued
    ▼
Secret `opencrvs-tls` (namespace: opencrvs-dev)
    │
    ▼
Traefik IngressRoutes → serve HTTPS using this secret
```

cert-manager issues one TLS certificate covering all subdomains (`opencrvs.com`, `login.opencrvs.com`, `auth.opencrvs.com`, etc.) via the **HTTP-01** challenge. No DNS API token needed.

---

## About Data Seeding

Data seeding is **fully automatic**:
1. Pre-install hook creates Kubernetes Secret `opencrvs-superuser` with a randomly generated password
2. Post-install hook runs the `data-seed` Job to seed all locations, forms, and users
3. `restartPolicy: OnFailure` — retries automatically if services aren't ready yet

---

## Prerequisites

| Requirement | Check |
|-------------|-------|
| Bare-metal server with public IP | Ports **80** and **443** must be open |
| Kubernetes installed (k3s, kubeadm, etc.) | `kubectl cluster-info` |
| Helm v3 | `helm version` |
| DNS control over `opencrvs.com` | A records pointing to this server |

---

## Step 1 — Open firewall ports

```bash
# Ubuntu / Debian
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw reload

# CentOS / RHEL
sudo firewall-cmd --permanent --add-port=80/tcp
sudo firewall-cmd --permanent --add-port=443/tcp
sudo firewall-cmd --reload
```

---

## Step 2 — Clone this repo on the server

```bash
git clone https://github.com/<your-username>/<your-repo>.git
cd <your-repo>
```

All commands below must be run from inside this directory.

---

## Step 3 — StorageClass (local-path-provisioner)

Provides persistent storage for databases.
Creates namespace `local-path-storage`.

```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.26/deploy/local-path-storage.yaml

kubectl patch storageclass local-path \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

Verify:
```bash
kubectl get storageclass
# Expected: local-path   rancher.io/local-path   (default)
```

---

## Step 4 — Install Traefik

Traefik acts as the ingress controller. It binds directly to ports 80 and 443 on the node.
Creates namespace `traefik`.

```bash
helm upgrade --install traefik oci://ghcr.io/traefik/helm/traefik \
  --namespace traefik \
  --create-namespace \
  -f traefik/values.yaml \
  --wait
```

Verify:
```bash
kubectl get pods -n traefik
# Expected: traefik-... Running
```

---

## Step 5 — Install cert-manager

cert-manager handles TLS certificate issuance from Let's Encrypt.
Creates namespace `cert-manager`.

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm upgrade --install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true \
  --wait
```

Verify:
```bash
kubectl get pods -n cert-manager
# Expected: cert-manager-..., cert-manager-cainjector-..., cert-manager-webhook-... all Running
```

---

## Step 6 — Point DNS to the server

Find your server's public IP:
```bash
curl -s https://ifconfig.me
```

In your DNS provider (`opencrvs.com`), create **two A records**:

| Type | Name | Value |
|------|------|-------|
| A | `@` (`opencrvs.com`) | `<SERVER-IP>` |
| A | `*` (`*.opencrvs.com`) | `<SERVER-IP>` |

Wait for propagation, then confirm all subdomains resolve:
```bash
dig opencrvs.com +short
dig login.opencrvs.com +short
dig auth.opencrvs.com +short
# All should return your server IP
```

> **DNS must be live before Step 7.** cert-manager uses the HTTP-01 challenge which requires your domains to be reachable on port 80.

---

## Step 7 — Apply ClusterIssuer and Certificate

First pre-create the `opencrvs-dev` namespace, then apply.

```bash
# Pre-create the namespace where the TLS secret will be stored
kubectl create namespace opencrvs-dev

# Create the Let's Encrypt ClusterIssuer
kubectl apply -f cert-manager/clusterissuer.yaml

# Verify ClusterIssuer is ready
kubectl get clusterissuer letsencrypt-prod
# READY should be True

# Request the TLS certificate
kubectl apply -f cert-manager/certificate.yaml
```

Wait for the certificate to be issued (takes 1–3 minutes):
```bash
kubectl get certificate opencrvs-cert -n opencrvs-dev -w
# Wait until READY = True
```

If it's taking long, check progress:
```bash
kubectl describe certificate opencrvs-cert -n opencrvs-dev
kubectl describe certificaterequest -n opencrvs-dev
```

> **Do not proceed to Step 8 until the certificate shows `READY = True`.**

---

## Step 8 — Install OpenCRVS Dependencies

Installs MongoDB, PostgreSQL, Elasticsearch, Redis, MinIO, and InfluxDB.
Creates namespace `opencrvs-deps-dev`.

```bash
helm upgrade --install opencrvs-deps oci://ghcr.io/opencrvs/opencrvs-dependencies-chart \
  --namespace opencrvs-deps-dev \
  --create-namespace \
  --atomic \
  --timeout 10m \
  -f dependencies/values.yaml
```

Wait for all pods:
```bash
kubectl get pods -n opencrvs-deps-dev
# All should show Running or Completed before continuing
```

---

## Step 9 — Install OpenCRVS

Installs all OpenCRVS application services into namespace `opencrvs-dev`.

The chart automatically:
- Creates the `opencrvs-superuser` Secret with a random admin password (pre-install hook)
- Deploys all services with TLS using the `opencrvs-tls` cert-manager secret
- Runs the `data-seed` Job (post-install hook)

```bash
helm upgrade --install opencrvs oci://ghcr.io/opencrvs/opencrvs-services \
  --timeout 1h \
  --namespace opencrvs-dev \
  --atomic \
  -f opencrvs-services/values.yaml
```

Monitor progress:
```bash
# Watch pods come up
kubectl get pods -n opencrvs-dev -w

# Watch data seed job
kubectl logs -f job/data-seed -n opencrvs-dev
```

---

## Step 10 — Verify

```bash
# All pods running?
kubectl get pods -n opencrvs-dev

# IngressRoutes created?
kubectl get ingressroute -n opencrvs-dev

# TLS certificate in place?
kubectl get secret opencrvs-tls -n opencrvs-dev

# Data seed completed?
kubectl get job data-seed -n opencrvs-dev
# COMPLETIONS should show 1/1
```

Open **`https://opencrvs.com`** — you should see the login page with a valid SSL certificate.

---

## Login

The admin password is auto-generated at install. Retrieve it:

```bash
kubectl get secret opencrvs-superuser -n opencrvs-dev \
  -o jsonpath='{.data.SUPER_USER_PASSWORD}' | base64 -d
echo
```

| Field | Value |
|-------|-------|
| URL | `https://opencrvs.com` |
| Username | `u.admin` |
| Password | *(output from command above)* |

---

## Upgrading

```bash
helm upgrade opencrvs oci://ghcr.io/opencrvs/opencrvs-services \
  --timeout 1h \
  --namespace opencrvs-dev \
  --atomic \
  -f opencrvs-services/values.yaml
```

> Data seed does **not** re-run on upgrade — only on fresh install.

---

## Troubleshooting

| Problem | Command / Fix |
|---------|--------------|
| Ports 80/443 not open | Run Step 1 firewall commands |
| Traefik pod `Pending` | `kubectl describe pod -n traefik` |
| cert-manager pods not starting | `kubectl describe pod -n cert-manager` |
| ClusterIssuer `READY: False` | `kubectl describe clusterissuer letsencrypt-prod` |
| Certificate not issuing | DNS not propagated yet — `dig opencrvs.com +short` |
| Certificate stuck `READY: False` | `kubectl describe certificaterequest -n opencrvs-dev` |
| Let's Encrypt rate limit hit | Change `server` in `cert-manager/clusterissuer.yaml` to `https://acme-staging-v02.api.letsencrypt.org/directory` (staging, no rate limits), re-apply |
| Dependency pods not starting | `kubectl describe pod <name> -n opencrvs-deps-dev` |
| OpenCRVS pods not starting | `kubectl describe pod <name> -n opencrvs-dev` |
| Data seed failed | `kubectl logs job/data-seed -n opencrvs-dev` |
| Data seed retrying (services not ready) | Normal — wait 5–10 minutes, it retries automatically |
| Need to re-run data seed | `kubectl delete job data-seed -n opencrvs-dev` then upgrade helm |
