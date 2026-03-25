---
title: "Building an Enterprise Platform on Bare Metal — Part 3: One Repo to Rule 84 Applications"
date: 2026-03-24
author: "Marius Oprin"
image: "/images/services/platform-engineering.png"
tags: ["kubernetes", "gitops", "argocd", "kargo", "helm", "series"]
draft: false
---

The hardest part of managing 84 applications isn't deploying them. It's keeping them from turning into a dumpster fire of configuration drift, secret sprawl, and "I swear I only changed one thing."

This is how we structured a single GitOps repository that manages everything — from Istio mesh configs to AI agent deployments — without losing our minds.

## The Repo Structure

Forget flat directories. At 84 apps, you need hierarchy or you'll drown:

```
gitops/
├── bootstrap/          # Tinkerbell + Ansible (Day 0)
├── catalog/
│   └── helm-charts/    # 11 custom charts we maintain
├── environments/
│   └── mgmt/           # Per-cluster values
├── mgmt-cluster/
│   └── apps/
│       ├── agentic/        # 17 apps — LiteLLM, Langfuse, agents, Open WebUI
│       ├── devops/         # 12 apps — ArgoCD, Argo Workflows, Harbor, Gitea, Kargo
│       ├── observability/  # 12 apps — Grafana, Mimir, Loki, Tempo, Sentry, Pyroscope
│       ├── security/       # 7 apps  — Vault, Falco, Kyverno, External Secrets
│       ├── identity/       # 4 apps  — Keycloak, CloudNativePG, Bank-Vaults
│       ├── infrastructure/ # 8 apps  — Traefik, cert-manager, MetalLB, Postfix
│       ├── service-mesh/   # 4 apps  — Istio, Kiali
│       ├── networking/     # 3 apps  — Tailscale
│       ├── storage/        # 2 apps  — Rook-Ceph operator + cluster
│       ├── collaboration/  # 3 apps  — Nextcloud, Synapse, n8n
│       ├── resilience/     # 3 apps  — VPA, descheduler, kured
│       └── backstage/      # 1 app   — Developer portal
└── scripts/                # Operational tooling
```

Every category has a `kustomization.yaml` that lists its ArgoCD Application manifests. A root app points to all categories. ArgoCD recurses from the top and discovers everything automatically.

## The Actual ArgoCD Application Manifest

Here's what a real app looks like in our repo — not a tutorial example, but how we deploy LiteLLM:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: litellm
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/BerriAI/litellm
    targetRevision: litellm-helm-v6.24.6
    path: deploy/charts/litellm-helm
    helm:
      valueFiles:
        - $values/environments/mgmt/agentic/litellm-values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: litellm
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

Key decisions:
- **`selfHeal: true`** everywhere. Non-negotiable. We learned this when someone ran `kubectl set resources` on Velero at 1 AM and ArgoCD reverted it in 3 seconds. Annoying in the moment, lifesaving long-term.
- **`ServerSideApply=true`** for apps with large CRDs (Istio, cert-manager). Without it, the `kubectl.kubernetes.io/last-applied-configuration` annotation exceeds the 256KB annotation limit and ArgoCD sync fails.
- **`CreateNamespace=true`** — let ArgoCD own namespace creation. Never pre-create namespaces manually.

## 11 Custom Helm Charts We Maintain

Not everything has a good upstream chart. We maintain our own for:

| Chart | Why |
|-------|-----|
| `capt-cluster` | Cluster API + Tinkerbell cluster definition |
| `openclaw` | OpenClaw gateway deployment |
| `mission-control` | Internal project tracker (Convex + Next.js) |
| `devops-ai-web` | DevOps AI platform frontend |
| `o8s-cloner` | Infrastructure cloning tool |
| `o8s-agents` | AI agent fleet deployment |
| `paperclip` | Document processing service |
| `synapse` | Matrix homeserver with custom config |
| `ops-dashboard` | Operations dashboard |
| `postfix` | SMTP relay (Postfix + iCloud SASL auth) |
| `common` | Shared templates and helpers |

Each chart lives in `catalog/helm-charts/` and is referenced by ArgoCD apps. We version them via git tags and use ArgoCD's multi-source feature to combine upstream charts with our custom values.

## The Velero Incident: Why Self-Heal Matters

2 AM. Velero's nodeAgent OOMKills during daily backups — second night in a row. The fix is simple: bump memory from 512Mi to 1Gi.

The wrong way (what we tried first):
```bash
kubectl set resources deployment/velero -n velero --limits=memory=1Gi
```

ArgoCD reverted it in 3 seconds flat. The deployment rolled out twice — once for our change, once for ArgoCD's revert. Net effect: nothing.

The right way:
```bash
# Edit the values file in the gitops repo
vim environments/mgmt/cluster-management/velero-values.yaml
# Change nodeAgent.resources.limits.memory: 1Gi
git commit -m "fix(velero): bump nodeAgent memory 512Mi→1Gi"
git push
```

ArgoCD synced within 30 seconds. Fixed permanently. Tracked in git history. No more OOMKills.

**This is the entire point of GitOps.** The pain of "I can't just kubectl it" pays off every single time something breaks at 2 AM and you need to know exactly what changed.

## Kargo: Promotion Pipelines

We use Kargo for version promotion across environments. When Harbor builds a new container image:

1. Kargo detects the new tag
2. Opens a promotion to update the gitops values
3. ArgoCD syncs the new version
4. If health checks fail, Kargo blocks further promotion

It's still early (Kargo is pre-1.0), but it already handles our most annoying workflow: "new image was pushed, now update 3 values files and make sure nothing breaks."

## What I'd Do Differently

**Start with app-of-apps from day one.** We initially created ArgoCD apps manually via the UI. By app #15, it was chaos. Migrating to app-of-apps retroactively meant recreating every Application as a YAML manifest and importing existing resources — a weekend of work that should have been avoided.

**Use ApplicationSets for repeated patterns.** We have 4 ApplicationSets for things deployed identically to every node (Alloy, kube-state-metrics, metrics-server, Prometheus CRDs). Should have used more.

**Pin Helm chart versions aggressively.** We had an incident where an unpinned chart auto-updated and broke Istio's mesh config. Now every `targetRevision` is explicit — no `HEAD`, no `main`, no `*`.

---

*Next: [Part 4](/blog/bare-metal-k8s-part4-observability/) — How we built multi-tenant observability with the LGTM stack.*
