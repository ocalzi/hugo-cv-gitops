# Architecture

## Overview

This platform runs a Hugo-based CV as a production Kubernetes workload on a single Hetzner Cloud node. The design prioritises:

- **Immutability** — Kairos OS prevents configuration drift on the node
- **GitOps** — all cluster state is declared in Git, ArgoCD is the single applier
- **CNCF-native** — standard Kubernetes APIs throughout (Gateway API, cert-manager, ArgoCD)
- **Real CI/CD** — not just `kubectl apply` — Octopus Deploy provides release tracking and deployment history

---

## Component Map

### Infrastructure Layer

| Component | Role | CNCF Project |
|-----------|------|-------------|
| [Kairos](https://kairos.io) | Immutable Linux OS, cloud-init bootstrap | Sandbox |
| [k3s](https://k3s.io) | Lightweight Kubernetes distribution | Sandbox |
| Hetzner CCM | Cloud Controller Manager — node IP, load balancer | — |

### Networking Layer

| Component | Role | CNCF Project |
|-----------|------|-------------|
| [Cilium](https://cilium.io) | CNI — pod networking, eBPF dataplane | Graduated |
| Cilium Gateway API | Ingress — replaces Traefik/nginx-ingress | Graduated |
| [Gateway API](https://gateway-api.sigs.k8s.io/) | Kubernetes ingress standard (SIG-network) | — |
| [CiliumLoadBalancerIPPool](https://docs.cilium.io/en/stable/network/lb-ipam/) | Pin Gateway to node public IP | Graduated |

### TLS Layer

| Component | Role | CNCF Project |
|-----------|------|-------------|
| [cert-manager](https://cert-manager.io) | Automated TLS via Let's Encrypt | Graduated |
| cert-manager-webhook-ovh | DNS-01 challenge solver for OVH | — |

### CI/CD Layer

| Component | Role |
|-----------|------|
| [Gitea](https://gitea.com) | Self-hosted Git SCM |
| Gitea Actions (act-runner) | CI runner — polls Gitea for jobs |
| Docker-in-Docker (dind) | Build daemon sidecar for act-runner |
| [Octopus Deploy](https://octopus.com) | Release orchestration and deployment tracking |
| [Argo CD](https://argoproj.github.io/cd/) | GitOps CD — syncs cluster to gitops repo |

### Application Layer

| Component | Role |
|-----------|------|
| Hugo | Static site generator |
| nginx | Web server inside container |
| Quay.io (or your registry) | Container image storage |

---

## Data Flow

### Build & Deploy (master push)

```
Developer pushes to Hugo-CV repo (master branch)
    │
    ▼
Gitea Actions picks up job (act-runner, self-hosted)
    │
    ├─ docker build (BuildKit/dind sidecar)
    ├─ docker push → quay.io/YOUR_REGISTRY/YOUR_IMAGE:master-<sha>
    └─ octopus release create --version <sha>
              │
              ▼
        Octopus Deploy
        ├─ "Deploy Kubernetes YAML" step
        │    → updates image tag in gitops repo
        └─ triggers ArgoCD sync (or ArgoCD polls on schedule)
              │
              ▼
           ArgoCD
           └─ syncs apps/hugo-cv → hugo-cv namespace
                   │
                   ▼
              Kubernetes
              ├─ cv-olivier   Deployment
              ├─ cv-architect Deployment
              └─ cv-octy      Deployment
```

### PR Preview

```
Developer opens PR in Hugo-CV repo
    │
    ▼
Gitea Actions
    ├─ docker build
    ├─ docker push → quay.io/.../cv:<branch>-<sha>
    └─ octopus release create (no deploy step)
              │
              ▼
    CI creates ArgoCD Application:
    name: hugo-cv-pr-<N>
    namespace: preview-hugo-cv-pr-<N>
    domain: pr-<N>.preview.YOUR_DOMAIN
              │
              ▼
    ArgoCD syncs → preview pod running
    URL: https://pr-<N>.preview.YOUR_DOMAIN
         (wildcard cert: *.preview.YOUR_DOMAIN)

PR closed → ArgoCD Application deleted → namespace pruned
```

---

## Multi-Persona Pattern

One Docker image, multiple distinct CV personas — each with their own domain, Deployment, Service, and HTTPRoute.

```
quay.io/YOUR_REGISTRY/YOUR_IMAGE:master-<sha>
    │
    ├─► cv-olivier    → Deployment → Service → HTTPRoute → cv.YOUR_DOMAIN
    ├─► cv-architect  → Deployment → Service → HTTPRoute → architect.YOUR_DOMAIN
    └─► cv-octy       → Deployment → Service → HTTPRoute → octy.YOUR_DOMAIN
```

The content per persona is controlled at the Hugo build level (themes, config, content) — the same image binary serves different content based on environment variables or baked-in content selection.

---

## Networking Detail

```
Internet
    │ (DNS → YOUR_NODE_IP)
    ▼
Hetzner Firewall (allows 80, 443)
    │
    ▼
Cilium Gateway (46.X.X.X:443)
CiliumLoadBalancerIPPool pins the external IP
    │
    ├─ SNI: cv.YOUR_DOMAIN          → https-cv listener    → cv-olivier Service
    ├─ SNI: *.preview.YOUR_DOMAIN   → https-preview listener → cv-preview Service
    ├─ SNI: argo.YOUR_DOMAIN        → https-argocd listener  → ArgoCD Service
    └─ SNI: git.YOUR_DOMAIN         → https-gitea listener   → Gitea Service

TLS terminated at the Gateway.
Backends receive plain HTTP on port 80.
```

---

## CNCF Project Links

- **Kubernetes**: https://kubernetes.io/docs/
- **k3s**: https://docs.k3s.io/
- **Kairos**: https://kairos.io/docs/
- **Cilium**: https://docs.cilium.io/en/stable/
- **Cilium Gateway API**: https://docs.cilium.io/en/stable/network/servicemesh/gateway-api/
- **cert-manager**: https://cert-manager.io/docs/
- **cert-manager DNS-01**: https://cert-manager.io/docs/configuration/acme/dns01/
- **Argo CD**: https://argo-cd.readthedocs.io/en/stable/
- **ArgoCD App-of-Apps**: https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/
- **Helm**: https://helm.sh/docs/
- **Gateway API**: https://gateway-api.sigs.k8s.io/guides/
