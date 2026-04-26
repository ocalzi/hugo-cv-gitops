# Security Considerations

This document covers the security posture of the `hugo-cv-gitops` platform — what is hardened, what is not, and what you should be aware of before deploying in your environment.

---

## Firewall

Configure your Hetzner Cloud Firewall (or equivalent) with the following inbound rules:

| Port | Protocol | Source | Purpose |
|------|----------|--------|---------|
| 22 | TCP | `YOUR_IP/32` | SSH — **restrict to your IP only** |
| 80 | TCP | `0.0.0.0/0` | HTTP (ACME challenges, HTTP→HTTPS redirects) |
| 443 | TCP | `0.0.0.0/0` | HTTPS (public CV, ArgoCD, Gitea) |
| 6443 | TCP | `YOUR_IP/32` | Kubernetes API — **restrict to your IP only** |

All other inbound traffic should be **dropped** by default.

> **Why restrict 22 and 6443?** Both ports give direct access to the node and the cluster. Exposing them to the internet invites brute-force and scanning. Restricting to your IP costs nothing and removes a large attack surface.

---

## SSH Hardening

The bootstrap configs (`bootstrap/cloud-config.yaml` and `bootstrap/cilium-cloud-config.yaml`) include:

```yaml
ssh_pwauth: false  # disable password auth — keys only
```

This disables password-based SSH login entirely. Only SSH keys listed in `ssh_authorized_keys` can authenticate.

SSH keys are imported from GitHub (`github:YOUR_GITHUB_USERNAME`) at boot time — no manual key copying required.

**Additional recommendations** (not implemented in this template):
- Change SSH port from 22 to a non-standard port (reduces log noise, not a security control)
- Install `fail2ban` or `sshguard` for brute-force protection if you open SSH to the internet

---

## Secrets Management

All secrets in this repo are provided as `*.example` files with placeholder values only. **Never commit actual secret values.**

| File | What it contains |
|------|-----------------|
| `platform/cert-manager/secret-ovh-credentials.yaml.example` | OVH API credentials for DNS-01 |
| `platform/argocd/secret-gitea-repo.yaml.example` | Gitea token for ArgoCD repo access |
| `platform/gitea-runner/secret-runner.yaml.example` | Gitea runner registration token |

**Workflow:**
1. Copy the `.example` file: `cp secret-foo.yaml.example /tmp/secret-foo.yaml`
2. Fill in actual values in `/tmp/` (outside the repo)
3. Apply: `kubectl apply -f /tmp/secret-foo.yaml`
4. Delete the local copy: `rm /tmp/secret-foo.yaml`

The `.gitignore` prevents common secret file patterns from being committed:

```
*.secret, *.env, *.key, *.pem, *.p12, *.pfx
*-credentials.yaml, *-credentials.json, *-secret.yaml
htpasswd, kubeconfig, kubeconfig.*
```

---

## What This Platform Does NOT Provide

Be explicit about the gaps before using this in production:

### No Kubernetes NetworkPolicies
By default, all pods in the cluster can communicate with each other. There is no network segmentation between namespaces. Cilium supports NetworkPolicies and CiliumNetworkPolicies — consider adding them if you run sensitive workloads alongside the CV.

### No mTLS Between Services
Service-to-service traffic inside the cluster is unencrypted. Cilium supports transparent mTLS via WireGuard encryption — not enabled in this template.

### No Web Application Firewall (WAF)
There is no WAF, rate limiting, or DDoS protection in front of the Cilium Gateway. The Gateway accepts all traffic that passes the Hetzner firewall.

### No Container Image Scanning
Built images are not scanned for vulnerabilities before deployment. Consider adding [Trivy Operator](https://aquasecurity.github.io/trivy-operator/) or a registry-level scan to your pipeline.

### No Runtime Security
There is no Falco or equivalent for detecting suspicious container behaviour at runtime.

### No Pod Security Standards
PodSecurityAdmission is not configured. Pods can request privileged mode (the Gitea runner dind sidecar does). Consider enforcing `restricted` or `baseline` standards on namespaces that don't need it.

---

## Single-Node Caveats

This platform runs on a single Kubernetes node. Security implications:

- **No pod isolation between nodes** — all pods share the same physical host, kernel, and network namespace stack
- **No etcd quorum** — k3s uses embedded SQLite; data loss on disk failure is possible
- **Node compromise = cluster compromise** — there is no separation between control plane and workloads
- **Maintenance = downtime** — node reboots (OS updates, kernel patches) bring everything down

This is acceptable for a personal CV platform. It is **not** suitable for multi-tenant workloads or sensitive data without significant additional hardening.

---

## Reporting Issues

If you find a security issue in this template, please open a [GitHub issue](https://github.com/ocalzi/hugo-cv-gitops/issues) or reach out directly.
