# hugo-cv-gitops

> Deploy a production-grade Hugo CV on Kubernetes using a full CNCF-native GitOps pipeline —
> with multi-persona support and PR preview environments. Running on Hetzner Cloud.

[![License](https://img.shields.io/badge/license-Apache%202.0-blue.svg)](LICENSE)
[![Kubernetes](https://img.shields.io/badge/kubernetes-CNCF%20Graduated-blue)](https://kubernetes.io)
[![ArgoCD](https://img.shields.io/badge/argocd-CNCF%20Graduated-blue)](https://argoproj.github.io/cd/)
[![Cilium](https://img.shields.io/badge/cilium-CNCF%20Graduated-orange)](https://cilium.io)

---

## Origin Story

This project started as a migration away from a setup that worked, but felt increasingly wrong.

The original stack was **Drone CI + Portainer** — a self-hosted CI pipeline pushing a Hugo CV to a Docker container managed through a web UI. It got the job done. But it wasn't something you could learn from, grow from, or be proud of showing.

The question became: what would it look like to rebuild this properly? Not just "make it work" — but use the tools the industry is actually converging on. GitOps instead of clicking in a UI. Kubernetes instead of bare Docker. Gateway API instead of a reverse proxy config file. cert-manager instead of manual certificate renewals.

The result is this repo. A Hugo CV — one of the simplest possible workloads — deployed through a full CNCF-native pipeline on an immutable OS, on a single Hetzner node. Deliberately over-engineered for a CV. Deliberately, because the point was never the CV.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Hetzner Cloud                               │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    Kairos Node (CPX42)                       │   │
│  │                                                              │   │
│  │  ┌──────────────┐   ┌──────────────┐   ┌────────────────┐   │   │
│  │  │    Kairos    │   │     k3s      │   │    Cilium      │   │   │
│  │  │ Immutable OS │   │  v1.35.2+k3s │   │ CNI + Gateway  │   │   │
│  │  └──────────────┘   └──────────────┘   └────────────────┘   │   │
│  │                                                              │   │
│  │  ┌──────────────┐   ┌──────────────┐   ┌────────────────┐   │   │
│  │  │ cert-manager │   │    ArgoCD    │   │  Gitea Runner  │   │   │
│  │  │  OVH DNS-01  │   │  GitOps CD   │   │ act + BuildKit │   │   │
│  │  └──────────────┘   └──────────────┘   └────────────────┘   │   │
│  │                                                              │   │
│  │  ┌──────────────┐   ┌──────────────┐   ┌────────────────┐   │   │
│  │  │   Gitea      │   │   Octopus    │   │   Hugo CV      │   │   │
│  │  │  SCM + CI    │   │    Deploy    │   │ nginx + multi  │   │   │
│  │  └──────────────┘   └──────────────┘   │    persona     │   │   │
│  │                                        └────────────────┘   │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Public IP → Hetzner Firewall → Cilium Gateway API (port 80/443)    │
└─────────────────────────────────────────────────────────────────────┘
```

---

## CNCF Stack

| Project | Role | CNCF Status | Docs |
|---------|------|-------------|------|
| [Kubernetes](https://kubernetes.io) | Container orchestration | Graduated | [Docs](https://kubernetes.io/docs/) |
| [k3s](https://k3s.io) | Lightweight Kubernetes distro | Sandbox | [Docs](https://docs.k3s.io/) |
| [Kairos](https://kairos.io) | Immutable Linux OS | Sandbox | [Docs](https://kairos.io/docs/) |
| [Cilium](https://cilium.io) | CNI + Gateway API implementation | Graduated | [Docs](https://docs.cilium.io/) |
| [cert-manager](https://cert-manager.io) | Automated TLS certificate management | Graduated | [Docs](https://cert-manager.io/docs/) |
| [Argo CD](https://argoproj.github.io/cd/) | GitOps continuous delivery | Graduated | [Docs](https://argo-cd.readthedocs.io/) |
| [Helm](https://helm.sh) | Kubernetes package management | Graduated | [Docs](https://helm.sh/docs/) |
| [Gateway API](https://gateway-api.sigs.k8s.io/) | Next-gen Kubernetes ingress (SIG-network) | — | [Docs](https://gateway-api.sigs.k8s.io/guides/) |

Plus: [Octopus Deploy](https://octopus.com) as release orchestrator, [Gitea](https://gitea.com) as self-hosted SCM.

---

## Features

- **Multi-persona CVs** — one Docker image, multiple deployments, multiple domains (`cv.`, `architect.`, etc.)
- **PR preview environments** — automatic `pr-N.preview.YOUR_DOMAIN` on every pull request
- **Wildcard TLS** — via DNS-01 challenge (OVH webhook for cert-manager)
- **In-cluster Docker builds** — BuildKit sidecar in Gitea Runner, no Docker Hub rate limits
- **Dual CI** — Gitea Actions (primary, self-hosted) + GitHub Actions (mirror, cloud-hosted)
- **Octopus Deploy** — release orchestration and deployment tracking between CI and ArgoCD
- **Immutable OS** — Kairos ensures reproducible, tamper-resistant node state

---

## Getting Started

See [docs/getting-started.md](docs/getting-started.md) for the full step-by-step guide.

**Prerequisites:**
- Hetzner Cloud account + API token
- Domain managed via OVH DNS (or adapt cert-manager issuer for your DNS provider)
- Octopus Deploy instance (self-hosted or cloud, free tier available)
- Quay.io or other container registry account
- Git provider — Gitea (self-hosted), GitHub, or GitLab. See [Gitea Helm chart](https://gitea.com/gitea/helm-chart) if you need a self-hosted instance

**Quick overview:**
1. Create a Hetzner server and boot Kairos from ISO
2. Upload `bootstrap/cloud-config.yaml` — k3s starts with CNI-ready flags
3. Install Cilium 1.18.7 and Hetzner CCM manually (see getting-started Step 4)
4. Apply platform manifests (`platform/`)
5. Configure ArgoCD to watch your gitops repo
6. Push your Hugo CV source — CI builds, Octopus deploys, ArgoCD syncs

---

## Pipeline

```
git push master (Hugo-CV repo)
        │
        ▼
┌───────────────────┐
│  Gitea Actions    │  (or GitHub Actions — same steps)
│  .gitea/workflows │  example workflows provided, adapt to your setup
│  /build.yml       │
│                   │
│  1. docker build  │──► YOUR_REGISTRY/YOUR_IMAGE:master-<sha>
│  2. docker push   │
│  3. octo release  │
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│  Octopus Deploy   │  ⚠ setup not covered here
│                   │  see octopus.com/docs
│  Project: Hugo-CV │
│  Env: production  │
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│     ArgoCD        │
│                   │
│  App: hugo-cv     │
│  → sync gitops    │
│    repo → cluster │
└────────┬──────────┘
         │
         ▼
  hugo-cv namespace
  ├── cv-<persona1> → cv.YOUR_DOMAIN
  └── cv-<persona2> → persona2.YOUR_DOMAIN
```

---

## Customisation

Replace these placeholders throughout the manifests:

| Placeholder | Your value | Example |
|------------|-----------|---------|
| `YOUR_DOMAIN` | Your root domain | `example.com` |
| `YOUR_PERSONA_NAME` | First CV persona name (used in Gateway listener and HTTPRoute) | `olivier` |
| `YOUR_NODE_IP` | Hetzner node public IP | `1.2.3.4` |
| `YOUR_HOSTNAME` | Kairos node hostname | `kairos` |
| `YOUR_REGISTRY` | Container registry host | `quay.io` |
| `YOUR_IMAGE` | Image name | `myuser/cv` |
| `YOUR_GITEA_URL` | Gitea base URL | `https://git.example.com` |
| `YOUR_ORG` | Gitea org/user | `my-org` |
| `YOUR_EMAIL` | ACME registration email | `admin@example.com` |
| `YOUR_GITHUB_USERNAME` | GitHub username for SSH key import | `myuser` |
| `YOUR_HETZNER_API_TOKEN` | Hetzner Cloud API token | — |
| `YOUR_OVH_APP_KEY` | OVH API application key | — |
| `YOUR_OVH_APP_SECRET` | OVH API application secret | — |
| `YOUR_OVH_CONSUMER_KEY` | OVH API consumer key | — |
| `YOUR_GITEA_TOKEN` | Gitea personal access token | — |
| `YOUR_GITEA_RUNNER_REGISTRATION_TOKEN` | Gitea runner token | — |

For password-protected CV (htpasswd): `htpasswd -B -c nginx/cv-htpasswd USERNAME`

---

## Issues & Lessons Learned

Real issues encountered building this platform. Shared so you don't have to rediscover them.

<details>
<summary>HTTPRoute backend returns 500 — cross-namespace ReferenceGrant missing</summary>

**Symptom:** The Gateway accepts the HTTPRoute and DNS resolves correctly, but requests return 500. The HTTPRoute status shows `RefNotPermitted`.

**Root cause:** Gateway API requires an explicit `ReferenceGrant` in the backend's namespace whenever an HTTPRoute references a Service in a different namespace. In this platform, HTTPRoutes live in `default` and Services live in `hugo-cv` — without the grant, Cilium rejects the backend reference entirely.

**Fix:** Create a `ReferenceGrant` in the `hugo-cv` namespace:
```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-default-to-hugo-cv
  namespace: hugo-cv
spec:
  from:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      namespace: default
  to:
    - group: ""
      kind: Service
```
This is included in `platform/gateway/referencegrant.yaml` and the Helm chart automatically creates it via `apps/hugo-cv/helm/templates/referencegrant.yaml`.

**Lesson:** Any time HTTPRoutes and their backend Services are in different namespaces, a ReferenceGrant is mandatory. It is not created automatically by any component.
</details>

<details>
<summary>HTTP-01 ACME challenge returns 404 on ALL routes, not just the challenge path</summary>

**Symptom:** cert-manager HTTP-01 challenge fails. Checking the challenge URL manually returns 404. But so do all other routes on the cluster — including ones that were working before.

**Root cause:** Cilium Envoy enters a degraded state when a Gateway listener references a TLS secret that does not yet exist. The Gateway continues to serve traffic but Envoy cannot program the listeners correctly, causing 404 across the board — not just on the broken listener.

**Fix:** Temporarily remove the HTTPS listener that references the missing secret from the Gateway spec. Allow the HTTP-01 challenge to complete and the secret to be created. Then restore the listener.

**Lesson:** On Cilium Gateway API, a single listener with a bad TLS secret reference poisons the entire Gateway. Don't add HTTPS listeners before their secrets exist.
</details>

<details>
<summary>HTTP-01 challenge intercepted by redirect HTTPRoute</summary>

**Symptom:** HTTP-01 ACME challenge returns 301 redirect instead of serving the challenge token. cert-manager reports the challenge as failed.

**Root cause:** A catch-all HTTPRoute with `PathPrefix: /` was redirecting all HTTP traffic to HTTPS — including the `/.well-known/acme-challenge/` path that cert-manager needs to serve over plain HTTP.

**Fix:** Temporarily delete the redirect HTTPRoute before triggering cert renewal. Restore it after the certificate is issued. Alternatively, switch to DNS-01 challenges to avoid this entirely.

**Lesson:** HTTP-01 and catch-all HTTP→HTTPS redirects don't coexist. DNS-01 avoids the problem entirely if your DNS provider is supported.
</details>

<details>
<summary>Octopus Kubernetes agent fails to poll — hairpin NAT</summary>

**Symptom:** Octopus K8s agent tentacle installed in-cluster, but it never connects to the Octopus server. Logs show connection timeouts to the external hostname.

**Root cause:** The agent was configured to poll the external Octopus hostname (`octopus.YOUR_DOMAIN`). On a single-node cluster without hairpin NAT, a pod cannot reach a LoadBalancer service via its external IP from inside the same node. The packet is dropped.

**Fix:** Configure the agent to use the internal cluster DNS name instead:
`http://octopus-deploy.octopus-deploy.svc.cluster.local:10943/`

**Lesson:** Always use internal cluster DNS for in-cluster service communication. External hostnames require hairpin NAT, which is often absent on single-node clusters.
</details>

<details>
<summary>Octopus Kubernetes agent PVC stuck in Pending — ReadWriteMany not supported</summary>

**Symptom:** Octopus agent Helm chart installs but the agent pod never starts. PVC stays in `Pending` state.

**Root cause:** The agent chart requests a `ReadWriteMany` (RWX) PersistentVolumeClaim by default. The `local-path` provisioner only supports `ReadWriteOnce` (RWO). The PVC can never be bound.

**Fix:** Install `csi-driver-nfs` (v4.10.0) and use it as the backing storage class:
```
--set persistence.nfs.backing.storageClassName=local-path
```

**Lesson:** Check your storage class access mode support before installing Helm charts that request RWX. `local-path` is RWO-only.
</details>

<details>
<summary>Octopus pre-install hook fails — ServiceAccount doesn't exist yet</summary>

**Symptom:** `helm install` for Octopus K8s agent fails immediately with a permissions error during the pre-install hook.

**Root cause:** The chart's pre-install hook runs a Job that needs a ServiceAccount (`octopus-agent-tentacle-pre`). But Helm creates resources in order, and the ServiceAccount hasn't been created yet when the hook fires.

**Fix:** Manually pre-create the ServiceAccount before running `helm install`:
```bash
kubectl create serviceaccount octopus-agent-tentacle-pre -n octopus-agent-production
```

**Lesson:** When a Helm chart's pre-install hook depends on resources that the chart itself creates, you may need to pre-create those resources manually.
</details>

<details>
<summary>Valkey (Redis cluster) quorum failure — session errors in Gitea</summary>

**Symptom:** Gitea returns session errors intermittently. Valkey (Redis-compatible) logs show cluster quorum failures.

**Root cause:** Valkey was configured with `cluster.nodes=1`. Redis cluster mode requires a minimum of 3 nodes to establish quorum. With 1 node, the cluster never becomes healthy.

**Fix:** Set `cluster.nodes=3` (minimum) and `cluster.replicas=0`:
```
helm upgrade gitea gitea-charts/gitea \
  --set valkey.cluster.nodes=3 \
  --set valkey.cluster.replicas=0
```

**Lesson:** Redis cluster mode has a hard minimum of 3 nodes. If you only need 1 Redis instance, disable cluster mode entirely and use standalone mode instead.
</details>

<details>
<summary>Hetzner CCM sets wrong node IP — kubelet TLS cert mismatch</summary>

**Symptom:** After installing Hetzner Cloud Controller Manager, nodes show a private IP as `InternalIP`. Components that rely on the node IP (like kubelet certificate SANs) break.

**Root cause:** CCM was installed with `networking.enabled=true`. When networking is enabled, the CCM activates its **Route Controller**, which programs routes into the Hetzner private network and sets the node's `InternalIP` to the **private network IP**. On a single-node cluster, this is unnecessary — and it breaks kubelet TLS because the kubelet certificate SAN only covers the public IP.

**What `networking.enabled` actually controls** ([official docs](https://github.com/hetznercloud/hcloud-cloud-controller-manager/blob/main/docs/explanation/private-networks.md)):
- `true` — enables the Route Controller: programs Hetzner private network routes for native pod-to-pod routing across nodes (no VXLAN overlay), and enables private IP targets for Hetzner LoadBalancers. Requires a CNI with native routing support (Cilium with `routing-mode: native`), a matching `clusterCIDR`, and all nodes attached to the private network.
- `false` (default) — Route Controller disabled. CNI handles all pod networking via overlay. Node `InternalIP` stays as the public IP. No IP range planning required.

**Fix:** Install CCM with `networking.enabled=false`:
```bash
helm upgrade --install hcloud-cloud-controller-manager \
  hetzner-cloud/hcloud-cloud-controller-manager \
  --namespace kube-system \
  --set networking.enabled=false
```

**Lesson:** On a single-node cluster there are no inter-node routes to program, so `networking.enabled=false` is always correct. Only enable it on multi-node clusters where you want native pod routing through Hetzner private networks — and only with a CNI configured for native routing mode.
</details>

<details>
<summary>Stale ACME challenges — cert-manager doesn't retry after 8+ hours</summary>

**Symptom:** A cert-manager challenge has been pending for many hours. The certificate is never issued. Deleting and recreating the Certificate resource doesn't help.

**Root cause:** cert-manager creates `Challenge` and `Order` resources. When a challenge fails, it is retried with exponential backoff — but after a certain point it stops retrying. Simply recreating the `Certificate` doesn't clean up the stuck `Order`.

**Fix:** Delete both the `Challenge` and the `Order` objects to force cert-manager to start fresh:
```bash
kubectl delete challenge -n default --all
kubectl delete order -n default --all
```
cert-manager will create new ones and begin the process again.

**Lesson:** When ACME challenges are stuck, delete Challenge and Order objects — not the Certificate.
</details>

<details>
<summary>cert-manager-webhook-ovh — missing RBAC, webhook cannot read credentials secret</summary>

**Symptom:** DNS-01 challenges stay pending. Webhook logs show:
```
secrets "letsencrypt-prod-dns01-ovh-credentials" is forbidden:
User "system:serviceaccount:cert-manager:cert-manager-webhook-ovh"
cannot get resource "secrets" in API group "" in the namespace "cert-manager"
```

**Root cause:** The webhook Helm chart creates no Role or RoleBinding for the credentials secret. The chart install notes warn about this ("For any Issuer or ClusterIssuer you wish to create, please ensure you also create the corresponding Role and RoleBinding") but provides no automation for it.

**Fix:** Manually create the Role and RoleBinding after installing the chart, then delete stuck challenges/orders to force a retry:
```bash
kubectl create role cert-manager-webhook-ovh-secret-reader \
  --namespace cert-manager \
  --verb=get \
  --resource=secrets \
  --resource-name=letsencrypt-prod-dns01-ovh-credentials

kubectl create rolebinding cert-manager-webhook-ovh-secret-reader \
  --namespace cert-manager \
  --role=cert-manager-webhook-ovh-secret-reader \
  --serviceaccount=cert-manager:cert-manager-webhook-ovh

kubectl delete challenges,orders -n default --all
```

**Lesson:** The webhook chart does not auto-create RBAC for your credentials secret. Always add the Role and RoleBinding immediately after installing the webhook, before applying ClusterIssuers.

> **Historical note:** cert-manager-webhook-ovh v0.9.8 previously used "ambient mode" — credentials were set at Helm install time via `--set ovhCredentials.*` and the ClusterIssuer secret refs were ignored. The current chart removed `ovhCredentials` entirely and requires credentials to be in a Kubernetes secret referenced by the ClusterIssuer. The ambient mode issue no longer applies.
</details>

<details>
<summary>Cilium 1.19.x regression — CNI broken, pods fail to start</summary>

**Symptom:** After upgrading Cilium to 1.19.x, new pods fail to start with network errors. Existing pods may be unaffected.

**Root cause:** Cilium 1.19.x introduced a regression that affects certain CNI configurations. Tracked in [cilium/cilium#44430](https://github.com/cilium/cilium/issues/44430).

**Fix:** Pin Cilium to 1.18.7:
```bash
cilium install --version 1.18.7
```

**Lesson:** Pin Cilium to a known-good version in production. Check the issue tracker before upgrading.
</details>

---

## Security Considerations

- SSH (TCP 22) and the Kubernetes API (TCP 6443) should be **restricted to your IP** in the Hetzner firewall — not open to the internet
- Password-based SSH auth is **disabled** in the bootstrap configs (`ssh_pwauth: false`) — keys only
- All `*.example` secret files contain placeholders only — never commit filled values
- This platform has **no WAF, no NetworkPolicies, no mTLS, no image scanning, no runtime security** — see [docs/security.md](docs/security.md) for the full posture and gap analysis

---

## Known Limitations

- **Single-node, no HA** — this setup runs everything on one node. No etcd quorum, no control-plane redundancy. A node reboot means brief downtime.
- **Hetzner-specific** — bootstrap config, CCM, and node assumptions are Hetzner-flavoured. Adapting to other cloud providers requires replacing the CCM and adjusting IP pool config.
- **OVH DNS-01 only** — cert-manager is configured for OVH DNS. Other DNS providers need a different webhook or solver. See [cert-manager DNS-01 providers](https://cert-manager.io/docs/configuration/acme/dns01/).
- **Cilium pinned to 1.18.7** — due to regression in 1.19.x ([cilium/cilium#44430](https://github.com/cilium/cilium/issues/44430)). Check issue status before upgrading.
- **Octopus Deploy licence required** — free tier available for small teams. See [octopus.com/pricing](https://octopus.com/pricing).
- **In-cluster BuildKit (Gitea Actions)** — the Gitea Actions workflow uses Docker-in-Docker inside the cluster. The GitHub Actions mirror uses Buildx on GitHub-hosted runners instead. Both push to the same external registry.

---

## Contributing

Issues and PRs welcome. If you hit a new problem not documented above, please open an issue — the Issues & Lessons Learned section is the most valuable part of this repo.

---

## License

Apache 2.0 — see [LICENSE](LICENSE).

---

[![AI assisted](https://img.shields.io/badge/AI%20assisted-Claude%20Code-blueviolet)](https://claude.ai/code)

Platform validation, divergence documentation, and manifest corrections in this repo were assisted by [Claude Code](https://claude.ai/code). All infrastructure decisions and real-cluster testing were performed by the author.
