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
- [ ] **Gitea** instance — self-hosted, or use an existing one
  - Runner registration token
  - Personal access token for ArgoCD
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

1. Copy `bootstrap/cilium-cloud-config.yaml` and fill in all `YOUR_*` placeholders.

2. During the Kairos install wizard (boot from ISO), paste your filled cloud-config when prompted, or upload it as the install configuration.

3. Kairos installs itself and reboots. On first boot, the `stages.network` block runs automatically:
   - Waits for k3s API
   - Installs Gateway API CRDs
   - Installs Cilium 1.18.7
   - Installs Hetzner CCM
   - Labels the node

4. Verify:
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

### cert-manager

```bash
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set installCRDs=true \
  --set gateway.enabled=true
```

Install OVH webhook:
```bash
helm repo add cert-manager-webhook-ovh https://aureq.github.io/cert-manager-webhook-ovh
helm install cert-manager-webhook-ovh cert-manager-webhook-ovh/cert-manager-webhook-ovh \
  --namespace cert-manager \
  --set groupName=acme.YOUR_DOMAIN \
  --set ovhCredentials.applicationKey=YOUR_OVH_APP_KEY \
  --set ovhCredentials.applicationSecret=YOUR_OVH_APP_SECRET \
  --set ovhCredentials.consumerKey=YOUR_OVH_CONSUMER_KEY
```

Create credentials secret:
```bash
cp platform/cert-manager/secret-ovh-credentials.yaml.example /tmp/secret-ovh-credentials.yaml
# Edit /tmp/secret-ovh-credentials.yaml with your OVH credentials
kubectl apply -f /tmp/secret-ovh-credentials.yaml
kubectl apply -f platform/cert-manager/clusterissuers.yaml
kubectl apply -f platform/cert-manager/certificate-cv.yaml
```

### Gateway

```bash
kubectl apply -f platform/gateway/main-gateway.yaml
kubectl apply -f platform/gateway/httproutes.yaml
```

Verify the Gateway gets the external IP:
```bash
kubectl get gateway main-gateway -n default
# ADDRESS column should show YOUR_NODE_IP
```

### ArgoCD

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd --namespace argocd --create-namespace
```

Add Gitea repo credentials:
```bash
cp platform/argocd/secret-gitea-repo.yaml.example /tmp/secret-gitea-repo.yaml
# Edit with your Gitea URL, username, and token
kubectl apply -f /tmp/secret-gitea-repo.yaml
```

Apply Hugo CV app:
```bash
kubectl apply -f platform/argocd/application-hugo-cv.yaml
```

### Gitea Runner

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

Create a `gitops` repository in your Gitea instance with the following structure:

```
gitops/
└── apps/
    └── hugo-cv/        ← copy apps/hugo-cv/helm/ here
        └── helm/
```

Update `values.yaml` with your domain and registry details.

---

## Step 6 — Configure CI Secrets

In your Hugo CV Gitea repo (Settings → Secrets → Actions), add:

| Secret | Value |
|--------|-------|
| `REGISTRY_USERNAME` | Your registry username |
| `REGISTRY_PASSWORD` | Your registry password/token |
| `OCTOPUS_URL` | `https://octopus.YOUR_DOMAIN` |
| `OCTOPUS_API_KEY` | Your Octopus API key |

---

## Step 7 — First Deploy

1. Push to `master` in your Hugo CV repo
2. Watch Gitea Actions run the build
3. Check Octopus Deploy for the release
4. Check ArgoCD for sync status
5. Visit `https://cv.YOUR_DOMAIN`

---

## Step 8 — Test PR Previews

1. Create a branch and open a PR
2. CI builds a preview image tagged `<branch>-<sha>`
3. Visit `https://pr-N.preview.YOUR_DOMAIN`
4. Close the PR — preview is automatically cleaned up

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
