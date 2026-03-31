# GitOps Cluster

ArgoCD-managed GitOps repository for the Kubernetes cluster at `kube.it.com`.

## Architecture

Three-layer sync-wave model. Each layer waits for the previous to be healthy before deploying.

```
root/               ArgoCD app-of-apps entry point
├── infrastructure/ [wave 1] Cluster primitives — networking, storage, load balancer
├── platform/       [wave 2] Shared services — cert-manager, external-secrets, operators
└── apps/           [wave 3] Workloads — RabbitMQ, MinIO, Keycloak, hello-world
```

### Infrastructure

| Component | Purpose | Namespace |
|-----------|---------|-----------|
| MetalLB | Bare-metal load balancer, IP `192.168.88.100` | `metallb-system` |
| Longhorn | Distributed block storage (default StorageClass) | `longhorn-system` |
| Cilium Gateway | Shared ingress gateway (`shared-gateway`) | `gateway` |

### Platform

| Component | Purpose | Namespace |
|-----------|---------|-----------|
| cert-manager | TLS certificate issuance via Let's Encrypt | `cert-manager` |
| external-secrets | Syncs secrets from Doppler into Kubernetes | `external-secrets` |
| RabbitMQ Operator | Manages `RabbitmqCluster` CRDs | `rabbitmq-system` |

### Apps

| App | URL | Notes |
|-----|-----|-------|
| RabbitMQ | `rabbit.kube.it.com` | 3-node HA cluster via Operator, management UI on :15672 |
| MinIO | `minio.kube.it.com` | Standalone, console UI on :9001 |
| Keycloak | `keycloak.kube.it.com` | Bundled PostgreSQL, `proxy: edge` |
| hello-world | `hello-world.kube.it.com` | nginx smoke-test app |

---

## Bootstrap — first-time cluster setup

These steps must be done **once manually** before ArgoCD can take over. Nothing in this repo replaces them.

### 1. Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 2. Apply the root app-of-apps

```bash
kubectl apply -k root/
```

ArgoCD will now deploy all three layers automatically in order.

### 3. Create the Doppler bootstrap secret

This is the **one secret that cannot come from Doppler itself** — it is the key that unlocks everything else.

Get a service token from: Doppler → your project → Access → Service Tokens

```bash
kubectl create secret generic doppler-token \
  --namespace external-secrets \
  --from-literal=dopplerToken=<your-doppler-service-token>
```

Once this secret exists, the `ClusterSecretStore` becomes ready and all `ExternalSecret` resources will sync automatically.

### 4. Add secrets to Doppler

In your Doppler project, create the following secrets:

| Key | Value | Used by |
|-----|-------|---------|
| `CLOUDFLARE_API_TOKEN` | Cloudflare API token with `Zone → DNS → Edit` on `kube.it.com` | cert-manager DNS-01 challenge |
| `KEYCLOAK_DB_PASSWORD` | Strong random password | Keycloak PostgreSQL database |

cert-manager will use `CLOUDFLARE_API_TOKEN` for DNS-01 challenges to issue the wildcard certificate `*.kube.it.com`.

### 5. Update the Let's Encrypt email

Replace the placeholder email in both ClusterIssuer files:

```
platform/cert-manager-config/clusterissuer-staging.yaml
platform/cert-manager-config/clusterissuer-prod.yaml
```

Change `your-email@example.com` to your real email. Let's Encrypt uses this for expiry notifications.

### 6. DNS — point the domain to the cluster

Add an A record for every subdomain pointing to the MetalLB IP `192.168.88.100`:

```
rabbit.kube.it.com      → 192.168.88.100
minio.kube.it.com       → 192.168.88.100
keycloak.kube.it.com    → 192.168.88.100
hello-world.kube.it.com → 192.168.88.100
```

Or use a wildcard record:

```
*.kube.it.com → 192.168.88.100
```

---

## TLS — how certificates work

```
Doppler
  └─ CLOUDFLARE_API_TOKEN
       └─ ExternalSecret → Secret: cloudflare-api-token (cert-manager ns)
            └─ ClusterIssuer: letsencrypt-prod
                 └─ Certificate: *.kube.it.com
                      └─ Secret: kube-it-com-tls (gateway ns)
                           └─ Gateway HTTPS listener (port 443)
```

The wildcard certificate covers all subdomains under `kube.it.com`. It is stored in the `gateway` namespace and terminated at the `shared-gateway`. No per-app TLS configuration is needed.

**Test with staging first** — use `letsencrypt-staging` issuer on the Certificate resource before switching to `letsencrypt-prod` to avoid hitting rate limits.

---

## RabbitMQ — credentials

The RabbitMQ Operator creates a default user secret automatically:

```bash
kubectl get secret rabbitmq-default-user -n rabbitmq -o jsonpath='{.data.username}' | base64 -d
kubectl get secret rabbitmq-default-user -n rabbitmq -o jsonpath='{.data.password}' | base64 -d
```

For production use, manage credentials via the Messaging Topology Operator:

```yaml
apiVersion: rabbitmq.com/v1beta1
kind: User
metadata:
  name: my-app
  namespace: rabbitmq
spec:
  rabbitmqClusterReference:
    name: rabbitmq
```

---

## Adding a new app

1. Create `apps/<app-name>/` with:
   - `namespace.yaml` — Namespace
   - `httproute.yaml` — HTTPRoute pointing to `shared-gateway` in `gateway` namespace
   - Any other manifests (Deployment, Service) or an `argocd-app.yaml` for Helm charts
   - `kustomization.yaml` — listing all of the above

2. Add `- <app-name>/` to `apps/kustomization.yaml`

3. Add a DNS record `<app>.kube.it.com → 192.168.88.100`

The wildcard TLS cert covers the new subdomain automatically — no cert changes needed.

---

## Useful commands

```bash
# Check ArgoCD app status
kubectl get applications -n argocd

# Watch cert issuance
kubectl describe certificate kube-it-com-wildcard -n gateway
kubectl describe certificaterequest -n gateway

# Check ExternalSecret sync status
kubectl get externalsecret -n cert-manager
kubectl get clustersecretstore

# RabbitMQ cluster health
kubectl get rabbitmqcluster -n rabbitmq
kubectl exec -n rabbitmq rabbitmq-server-0 -- rabbitmq-diagnostics cluster_status

# Force ArgoCD to sync immediately
argocd app sync <app-name>
```