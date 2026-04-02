# GitOps Cluster

ArgoCD-managed GitOps repository for the Kubernetes cluster at `kube.it.com`.

## Architecture

Three-layer sync-wave model. Each layer waits for the previous to be healthy before deploying.

```
root/               ArgoCD app-of-apps entry point
├── infrastructure/ [wave 1] Cluster primitives + infrastructure services
├── platform/       [wave 2] Shared services — cert-manager, external-secrets, operators, DDNS
└── apps/           [wave 3] Workloads — hello-world
```

### Infrastructure

| Component | Purpose | Namespace |
|-----------|---------|-----------|
| MetalLB | Bare-metal load balancer, IP `192.168.88.100` | `metallb-system` |
| Longhorn | Distributed block storage (default StorageClass) | `longhorn-system` |
| Cilium Gateway | Shared ingress gateway (`shared-gateway`) | `gateway` |
| RabbitMQ | 3-node HA cluster via Operator | `rabbitmq` |
| MinIO | Object storage, standalone | `minio` |
| Keycloak | Identity provider, bundled PostgreSQL | `keycloak-system` |
| Vault | HashiCorp secrets management | `vault` |

### Platform

| Component | Purpose | Namespace |
|-----------|---------|-----------|
| cert-manager | TLS certificate issuance via Let's Encrypt | `cert-manager` |
| external-secrets | Syncs secrets from Doppler into Kubernetes | `external-secrets` |
| RabbitMQ Operator | Manages `RabbitmqCluster` CRDs | `rabbitmq-system` |
| Keycloak Operator | Manages `Keycloak` CRDs | `keycloak-system` |
| cloudflare-ddns | Keeps Cloudflare DNS A records in sync with public IP | `cloudflare-ddns` |
| ArgoCD config | `argocd-cm` / `argocd-cmd-params-cm` patches | `argocd` |

### Apps

| App | URL | Notes |
|-----|-----|-------|
| hello-world | `hello-world.kube.it.com` | nginx smoke-test app |

### Services and their URLs

| Service | URL |
|---------|-----|
| ArgoCD | `argocd.kube.it.com` |
| Longhorn | `longhorn.kube.it.com` |
| RabbitMQ | `rabbit.kube.it.com` |
| MinIO | `minio.kube.it.com` |
| Keycloak | `keycloak.kube.it.com` |
| Vault | `vault.kube.it.com` |

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

Get a service token from: Doppler → your project → Access → Service Tokens. The token needs **read and write permissions** so that PushSecrets can write auto-generated credentials back.

```bash
kubectl create secret generic doppler-token \
  --namespace external-secrets \
  --from-literal=dopplerToken=<your-doppler-service-token>
```

Once this secret exists, the `ClusterSecretStore` becomes ready and all `ExternalSecret` resources will sync automatically.

### 4. Add secrets to Doppler

In your Doppler project, create the following secrets before ArgoCD reaches the platform wave:

| Key | Value | Used by |
|-----|-------|---------|
| `CLOUDFLARE_API_TOKEN` | Cloudflare API token with `Zone → DNS → Edit` on `kube.it.com` | cert-manager DNS-01 + cloudflare-ddns |
| `KEYCLOAK_DB_PASSWORD` | Strong random password | Keycloak PostgreSQL database |

The following secrets are **written back to Doppler automatically** by PushSecrets once the services are running:

| Key | Written by |
|-----|-----------|
| `ARGOCD_ADMIN_PASSWORD` | ArgoCD initial admin secret |
| `RABBITMQ_DEFAULT_USERNAME` | RabbitMQ Operator |
| `RABBITMQ_DEFAULT_PASSWORD` | RabbitMQ Operator |
| `RABBITMQ_HOST` | RabbitMQ Operator |
| `RABBITMQ_PORT` | RabbitMQ Operator |
| `MINIO_ROOT_USER` | MinIO Helm chart |
| `MINIO_ROOT_PASSWORD` | MinIO Helm chart |
| `KEYCLOAK_ADMIN_USERNAME` | Keycloak Operator |
| `KEYCLOAK_ADMIN_PASSWORD` | Keycloak Operator |

### 5. DNS — Cloudflare setup

DNS records are managed automatically by `cloudflare-ddns` once deployed. It keeps all subdomain A records pointed at your current public IP and refreshes every 10 seconds.

For the initial bootstrap (before cloudflare-ddns is running), you can manually add a wildcard A record in Cloudflare:

```
*.kube.it.com → <your public IP>
```

**Important Cloudflare settings:**
- SSL/TLS mode must be set to **Full (Strict)** — Cloudflare proxies the connection and forwards it to the origin on HTTPS port 443
- The Cilium gateway terminates TLS using the Let's Encrypt wildcard cert; Cloudflare handles client-facing TLS separately
- Port forwarding on the router: `443 → 192.168.88.100:443`

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

## Vault — initial setup

Vault deploys in a sealed state and requires one-time manual initialization:

```bash
# Initialize Vault (generates unseal keys + root token)
kubectl exec -n vault vault-0 -- vault operator init

# Unseal (run 3 times with different keys from the output above)
kubectl exec -n vault vault-0 -- vault operator unseal <unseal-key>
```

Save the unseal keys and root token in Doppler immediately after initialization.

---

## RabbitMQ — credentials

The RabbitMQ Operator creates a default user secret automatically. Credentials are also pushed to Doppler via PushSecret:

```bash
kubectl get secret rabbitmq-default-user -n rabbitmq -o jsonpath='{.data.username}' | base64 -d
kubectl get secret rabbitmq-default-user -n rabbitmq -o jsonpath='{.data.password}' | base64 -d
```

---

## Adding a new infrastructure service

1. Create `infrastructure/<service-name>/` with:
   - `argocd-app.yaml` — ArgoCD Application (for Helm charts) or raw manifests
   - `httproute.yaml` — HTTPRoute pointing to `shared-gateway` in `gateway` namespace
   - `kustomization.yaml` — listing all of the above

2. Add `- <service-name>/` to `infrastructure/kustomization.yaml`

3. Add the subdomain to `DOMAINS` in `platform/cloudflare-ddns/deployment.yaml`

The wildcard TLS cert covers the new subdomain automatically — no cert changes needed.

## Adding a new app

1. Create `apps/<app-name>/` with:
   - `namespace.yaml` — Namespace
   - `httproute.yaml` — HTTPRoute pointing to `shared-gateway` in `gateway` namespace
   - Any other manifests or an `argocd-app.yaml` for Helm charts
   - `kustomization.yaml` — listing all of the above

2. Add `- <app-name>/` to `apps/kustomization.yaml`

3. Add the subdomain to `DOMAINS` in `platform/cloudflare-ddns/deployment.yaml`

---

## Useful commands

```bash
# Check ArgoCD app status
kubectl get applications -n argocd

# Watch cert issuance
kubectl describe certificate kube-it-com-wildcard -n gateway
kubectl describe certificaterequest -n gateway

# Check ExternalSecret sync status
kubectl get externalsecret -A
kubectl get clustersecretstore

# Check PushSecret status
kubectl get pushsecret -A

# RabbitMQ cluster health
kubectl get rabbitmqcluster -n rabbitmq
kubectl exec -n rabbitmq rabbitmq-server-0 -- rabbitmq-diagnostics cluster_status

# Vault status
kubectl exec -n vault vault-0 -- vault status

# Force ArgoCD to sync immediately
argocd app sync <app-name>

# Restart ArgoCD server (e.g. after config changes)
kubectl rollout restart deployment argocd-server -n argocd
```
