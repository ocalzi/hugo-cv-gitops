# Getting Started

This guide takes you from a blank Hetzner server to a fully operational Hugo CV platform with GitOps pipeline.

## Prerequisites

Before starting, you need:

- [ ] **Hetzner Cloud account** — [console.hetzner.cloud](https://console.hetzner.cloud)
  - API token (Read & Write)
  - Server type: CPX21 minimum, CPX42 recommended (16 vCPU / 32 GB)
- [ ] **Domain** managed via OVH DNS — [api.ovh.com/createApp](https://api.ovh.com/createApp/)
  - OVH API credentials (applicationKey, applicationSecret, consumerKey)
  - DNS rights: GET/POST/PUT on `/domain/zone/*`
- [ ] **Container registry** — Quay.io, Docker Hub, GHCR, or self-hosted
  - Push credentials
- [ ] **Octopus Deploy** — [octopus.com](https://octopus.com) (self-hosted or cloud, free tier OK)
  - API key
  - Project named `Hugo-CV` in `Spaces-1`
- [ ] **Git provider** — Gitea (self-hosted), GitHub, or GitLab
  - A repository to host your Hugo CV source
  - A repository (or path) to act as the ArgoCD gitops source
  - Personal access token for ArgoCD repository access
  - If using Gitea Actions or GitHub Actions: runner registration token (Gitea) or Actions enabled (GitHub)
- [ ] **GitHub account** — for SSH key import (`github:YOUR_USERNAME` in cloud-config)
- [ ] Local tools: `kubectl`, `helm`, `cilium` CLI

---

## Step 1 — Provision Hetzner Server

1. In Hetzner Cloud Console, create a server:
   - OS: **Custom image** → upload Kairos ISO (download from [kairos.io/releases](https://kairos.io/releases))
   - Type: CPX42 (or smaller for testing)
   - Location: your preference
   - **No SSH key** at creation — Kairos cloud-init handles it

2. In the Hetzner Firewall, allow inbound:

   | Port | Protocol | Source | Purpose |
   |------|----------|--------|---------|
   | 22 | TCP | `YOUR_IP/32` | SSH — restrict to your IP only |
   | 80 | TCP | `0.0.0.0/0` | HTTP (ACME challenges, redirects) |
   | 443 | TCP | `0.0.0.0/0` | HTTPS (public CV, ArgoCD, Gitea) |
   | 6443 | TCP | `YOUR_IP/32` | Kubernetes API — restrict to your IP only |

   All other inbound traffic should be **dropped** by default.

3. Note the server's public IP → set as `YOUR_NODE_IP`

---

## Step 2 — Bootstrap Kairos

1. Copy `bootstrap/cloud-config.yaml` and fill in all `YOUR_*` placeholders.

2. During the Kairos install wizard (boot from ISO), paste your filled cloud-config when prompted, or upload it as the install configuration.

3. Kairos installs itself and reboots with k3s running. Cilium and Hetzner CCM are **not** installed automatically — you will install them manually in Step 4.

4. Verify k3s is up (node will show `NotReady` until Cilium is installed — that is expected):
   ```bash
   ssh kairos@YOUR_NODE_IP
   kubectl get nodes
   kubectl -n kube-system get pods | grep cilium
   ```

---

## Step 3 — Configure kubectl locally

```bash
scp kairos@YOUR_NODE_IP:/etc/rancher/k3s/k3s.yaml ~/.kube/hugo-cv-config
# Replace 127.0.0.1 with YOUR_NODE_IP in the file
export KUBECONFIG=~/.kube/hugo-cv-config
kubectl get nodes
```

---

## Step 4 — Apply Platform Manifests

### Cilium & Hetzner CCM

Install Gateway API experimental CRDs first — Cilium requires them:
```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/experimental-install.yaml
```

Install Cilium (pinned to 1.18.7 — do **not** use 1.19.x, see [cilium/cilium#44430](https://github.com/cilium/cilium/issues/44430)):
```bash
# operator.replicas=1 is required on single-node clusters
cilium install --version 1.18.7 \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost=127.0.0.1 \
  --set k8sServicePort=6443 \
  --set gatewayAPI.enabled=true \
  --set gatewayAPI.secretsNamespace.create=true \
  --set operator.replicas=1 \
  --set ipam.mode=kubernetes \
  --set hubble.enabled=true \
  --set hubble.relay.enabled=true
```

> **Note:** `cilium install --wait` may time out with Hubble relay pending. This is expected — the `node.cloudprovider.kubernetes.io/uninitialized` taint blocks scheduling until CCM is installed. Proceed to CCM install.

Create the Hetzner CCM credentials secret (API token from Hetzner Console → Security → API Tokens):
```bash
kubectl create secret generic hcloud -n kube-system \
  --from-literal=token="YOUR_HETZNER_API_TOKEN" \
  --from-literal=network="YOUR_HETZNER_NETWORK_NAME"
```

Install Hetzner Cloud Controller Manager:
```bash
helm repo add hetzner-cloud https://charts.hetzner.cloud
helm upgrade --install hcloud-cloud-controller-manager \
  hetzner-cloud/hcloud-cloud-controller-manager \
  --namespace kube-system \
  --set networking.enabled=false
```

> **Why `networking.enabled=false`?** On a single-node cluster there are no inter-node routes to program. Enabling it causes the CCM to replace the node's `InternalIP` with the Hetzner private network IP, which breaks kubelet TLS certificate validation.

Verify the node is ready and the uninitialized taint is cleared:
```bash
kubectl get nodes          # STATUS = Ready
kubectl -n kube-system get pods | grep cilium
kubectl -n kube-system get pods | grep hcloud
```

### cert-manager

```bash
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set crds.enabled=true
```

> **Note:** `--set gateway.enabled=true` was removed in cert-manager v1.15+. Gateway API support is now BETA and enabled by default (`ExperimentalGatewayAPISupport=true`). The old `--set installCRDs=true` flag still works but is deprecated — use `crds.enabled` instead.

Install OVH webhook:
```bash
helm repo add cert-manager-webhook-ovh https://aureq.github.io/cert-manager-webhook-ovh
helm install cert-manager-webhook-ovh cert-manager-webhook-ovh/cert-manager-webhook-ovh \
  --namespace cert-manager \
  --set groupName=acme.YOUR_DOMAIN
```

> **Note:** The old `--set ovhCredentials.*` flags no longer exist in the chart schema. Credentials are now passed via a Kubernetes secret referenced in the ClusterIssuer (see below).

Create credentials secret and grant the webhook RBAC access to read it:
```bash
cp platform/cert-manager/secret-ovh-credentials.yaml.example /tmp/secret-ovh-credentials.yaml
# Edit /tmp/secret-ovh-credentials.yaml with your OVH credentials
kubectl apply -f /tmp/secret-ovh-credentials.yaml

# Grant the webhook ServiceAccount permission to read the secret
kubectl create role cert-manager-webhook-ovh-secret-reader \
  --namespace cert-manager \
  --verb=get \
  --resource=secrets \
  --resource-name=letsencrypt-prod-dns01-ovh-credentials
kubectl create rolebinding cert-manager-webhook-ovh-secret-reader \
  --namespace cert-manager \
  --role=cert-manager-webhook-ovh-secret-reader \
  --serviceaccount=cert-manager:cert-manager-webhook-ovh

kubectl apply -f platform/cert-manager/clusterissuers.yaml
kubectl apply -f platform/cert-manager/certificate-cv.yaml
```

### Gateway

```bash
kubectl apply -f platform/gateway/main-gateway.yaml
kubectl apply -f platform/gateway/httproutes.yaml
kubectl apply -f platform/gateway/referencegrant.yaml
```

> **Why the ReferenceGrant?** HTTPRoutes live in the `default` namespace but reference Services in the `hugo-cv` namespace. Gateway API requires an explicit `ReferenceGrant` in the target namespace to permit cross-namespace references. Without it, Cilium rejects the route with `RefNotPermitted` and the backend returns 500.

Verify the Gateway gets the external IP:
```bash
kubectl get gateway main-gateway -n default
# ADDRESS column should show YOUR_NODE_IP
```

### Git Provider

ArgoCD can sync from any Git provider — Gitea, GitHub, or GitLab. The CI workflows (`.gitea/workflows/` and `.github/workflows/`) cover both Gitea Actions and GitHub Actions; use whichever matches your provider.

**If you need a self-hosted Git instance**, Gitea is the recommended option. Deploy it via the official Helm chart:
```bash
helm repo add gitea-charts https://dl.gitea.com/charts/
helm repo update
helm install gitea gitea-charts/gitea --namespace gitea --create-namespace
```
See the [Gitea Helm chart documentation](https://gitea.com/gitea/helm-chart) for full configuration options, including ingress, storage, and the Valkey (Redis) session backend.

> **Valkey note:** The Gitea chart defaults to `cluster.nodes=1` for Valkey, which causes quorum failures (Redis cluster requires a minimum of 3 nodes). Either set `cluster.nodes=3, cluster.replicas=0` or disable cluster mode entirely for a single-node setup. See the Issues & Lessons Learned section in the README.

**Public repos** (GitHub/GitLab) require no credentials secret in ArgoCD. For private repos or self-hosted Gitea, create a repository secret:
```bash
cp platform/argocd/secret-gitea-repo.yaml.example /tmp/secret-repo.yaml
# Edit with your Git provider URL, username, and token
kubectl apply -f /tmp/secret-repo.yaml
```

### ArgoCD

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd --namespace argocd --create-namespace
```

Apply the Hugo CV Application (update `repoURL` in the manifest to point at your gitops repo):
```bash
kubectl apply -f platform/argocd/application-hugo-cv.yaml
```

### Gitea Runner (optional — skip if using GitHub Actions)

```bash
kubectl apply -f platform/gitea-runner/namespace.yaml
kubectl apply -f platform/gitea-runner/pvc.yaml
cp platform/gitea-runner/secret-runner.yaml.example /tmp/secret-runner.yaml
# Edit with your Gitea runner registration token
kubectl apply -f /tmp/secret-runner.yaml
kubectl apply -f platform/gitea-runner/deployment.yaml
```

---

## Step 5 — Set Up Your gitops Repository

Create a repository (on Gitea, GitHub, or GitLab) with the following structure:

```
gitops/
└── apps/
    └── hugo-cv/        ← copy apps/hugo-cv/helm/ here
        └── helm/
```

Update `values.yaml` with your domain and registry details. Point the `repoURL` in `platform/argocd/application-hugo-cv.yaml` at this repository.

---

## Step 6 — CI Pipeline (Reference)

This repo includes example CI workflows for building and publishing the Hugo CV image:

- `.gitea/workflows/build.yml` — Gitea Actions (uses dind sidecar, in-cluster BuildKit)
- `.github/workflows/build.yml` — GitHub Actions (uses `docker/build-push-action` with Buildx)

Both workflows follow the same logic:
- `master` push → build image tagged `master-<sha>` → push to registry → trigger Octopus Deploy release
- PR open → build image tagged `<branch>-<sha>` → push to registry (no deploy)
- PR close → clean up preview ArgoCD Application

These are provided as **examples**. Adapt them to your registry, branching model, and release tooling.

### Octopus Deploy

The workflows use [Octopus Deploy](https://octopus.com/docs) to create a release and trigger deployment after a successful image build. Octopus setup (spaces, projects, deployment targets) is outside the scope of this guide — refer to the [Octopus Deploy documentation](https://octopus.com/docs) for setup instructions.

The required CI secrets in your Hugo CV repository are:

| Secret | Value |
|--------|-------|
| `REGISTRY_USERNAME` | Container registry username |
| `REGISTRY_PASSWORD` | Container registry password/token |
| `OCTOPUS_URL` | Your Octopus Deploy server URL |
| `OCTOPUS_API_KEY` | Octopus API key |

---

## Troubleshooting

**Gateway not getting external IP:**
- Check `CiliumLoadBalancerIPPool` is applied and `YOUR_NODE_IP` matches the server IP
- Check Cilium operator logs: `kubectl logs -n kube-system -l app.kubernetes.io/name=cilium-operator`

**ACME challenge failing:**
- Check for stuck Challenge/Order objects: `kubectl get challenges,orders -A`
- Delete stuck objects to force retry: `kubectl delete challenges,orders -A --all`
- Verify OVH credentials: check webhook pod logs in `cert-manager` namespace

**Pods not starting:**
- Check image pull secret is created in the right namespace
- Check registry credentials

**ArgoCD not syncing:**
- Verify repo secret is correct: `kubectl get secret -n argocd`
- Check ArgoCD repo server logs
