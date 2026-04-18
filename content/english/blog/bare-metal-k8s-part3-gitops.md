---
title: "Building an Enterprise Platform on Bare Metal — Part 3: One Repo to Rule 84 Applications"
date: 2026-03-24
author: "Marius Oprin"
image: "/images/services/platform-engineering.png"
tags: ["kubernetes", "gitops", "argocd", "kargo", "helm", "series"]
draft: false
description: "How we structure a single GitOps repository to manage 84 production Kubernetes applications — the directory layout, the real Argo CD manifests, the incidents that shaped the patterns, and what we would do differently."
summary: "The hardest part of managing 84 applications isn't deploying them — it's stopping them from turning into a dumpster fire of configuration drift. Here is the repo layout, the real Argo CD manifests, and the incidents that shaped the patterns."
---

The hardest part of managing 84 applications isn't deploying them. It's keeping them from turning into a dumpster fire of configuration drift, secret sprawl, and "I swear I only changed one thing."

This is how we structured a single GitOps repository that manages everything on the cluster — from Istio mesh configs to AI agent deployments — without losing our minds.

## The Repo Structure

Forget flat directories. At 84 apps you need hierarchy, or you drown in a `ls` of `apps/`:

```
gitops/
├── bootstrap/          # Tinkerbell + CAPI + Ansible (Day 0)
├── catalog/
│   └── helm-charts/    # 9 custom charts we maintain
├── environments/
│   └── mgmt/           # Per-cluster values
├── mgmt-cluster/
│   └── apps/
│       ├── agentic/        # 21 apps — 18 AI agents + LiteLLM + Langfuse + Open WebUI
│       ├── devops/         # 12 apps — Argo CD, Argo Workflows, Harbor, Gitea, Kargo
│       ├── observability/  # 12 apps — Grafana, Mimir, Loki, Tempo, Sentry, Pyroscope
│       ├── security/       #  7 apps — Vault, Falco, Kyverno, External Secrets
│       ├── identity/       #  4 apps — Keycloak, CloudNativePG, Bank-Vaults
│       ├── infrastructure/ #  8 apps — Traefik, cert-manager, MetalLB, Postfix
│       ├── service-mesh/   #  4 apps — Istio, Kiali
│       ├── networking/     #  3 apps — Tailscale
│       ├── storage/        #  2 apps — Rook-Ceph operator + cluster
│       ├── collaboration/  #  3 apps — Nextcloud, Synapse, n8n
│       ├── resilience/     #  3 apps — VPA, descheduler, kured
│       └── backstage/      #  1 app  — Developer portal
└── scripts/                # Operational tooling
```

That directory tree covers 80 applications. The remaining four are cluster-wide DaemonSets deployed through Argo CD ApplicationSets — Alloy, kube-state-metrics, metrics-server, and the Prometheus CRDs — which don't have a natural home in any single category. Total: **84**.

Every category has a `kustomization.yaml` that lists its Argo CD `Application` manifests. A root app-of-apps points to all categories. Argo CD recurses from the top and discovers everything automatically.

## A Real Argo CD Application

Here is what an app actually looks like in the repo — not a tutorial example, but how we deploy LiteLLM:

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

Three decisions in there earned their place the hard way:

- **`selfHeal: true`** on every app. Non-negotiable. We learned this when someone ran `kubectl set resources` on Velero at 1 AM and Argo CD reverted it in three seconds. Annoying in the moment, lifesaving long-term.
- **`ServerSideApply=true`** for apps with large CRDs (Istio, cert-manager, Traefik). Without it, the `last-applied-configuration` annotation blows past the 256 KB limit and the sync fails with a cryptic error about annotation size.
- **`CreateNamespace=true`** — Argo CD owns namespace creation. No pre-creating namespaces by hand, no drift between cluster state and repo state.

## Nine Custom Helm Charts

Not everything has a good upstream chart. We maintain our own for:

| Chart | Why |
| --- | --- |
| `capt-cluster` | Cluster API + Tinkerbell cluster definition |
| `mission-control` | Internal project tracker (Convex + Next.js) |
| `devops-ai-web` | DevOps AI platform frontend |
| `o8s-cloner` | Infrastructure cloning tool |
| `o8s-agents` | AI agent fleet deployment |
| `synapse` | Matrix homeserver with our SSO + federation config |
| `ops-dashboard` | Operations dashboard |
| `postfix` | SMTP relay (Postfix + iCloud SASL auth) |
| `common` | Shared templates and helpers used by the others |

Each chart lives in `catalog/helm-charts/` and is referenced by Argo CD apps. We version them via git tags and use Argo CD's multi-source feature to combine upstream charts with our custom values.

## The Velero Incident: Why Self-Heal Matters

2 AM. Velero's node agent OOMKills during the nightly backup — second night in a row. The fix is obvious: bump memory from 512 Mi to 1 Gi.

The wrong way — what we tried first:

```bash
kubectl set resources deployment/velero -n velero --limits=memory=1Gi
```

Argo CD reverted it in three seconds flat. The deployment rolled out twice — once for our change, once for Argo CD's revert. Net effect: nothing, plus two rollouts worth of log noise.

The right way:

```bash
vim environments/mgmt/cluster-management/velero-values.yaml
# nodeAgent.resources.limits.memory: 1Gi
git commit -m "fix(velero): bump nodeAgent memory 512Mi→1Gi"
git push
```

Argo CD synced within thirty seconds. Fixed permanently. Tracked in git history. No more OOMKills.

This is the entire point of GitOps. The pain of "I can't just `kubectl` it" pays off every single time something breaks at 2 AM and you need to know exactly what changed.

## Kargo: Promotion Pipelines

We use Kargo for version promotion across environments. When Harbor builds a new container image:

1. Kargo detects the new tag via a `Warehouse` subscription.
2. It opens a `Promotion` that rewrites the tag in the GitOps values file.
3. Argo CD syncs the new version into the target `Stage`.
4. If health checks fail, Kargo blocks further promotion and rolls back.

Kargo is still pre-1.0 and has sharp edges, but it already handles our most tedious workflow: "a new image got pushed, now update three values files and make sure nothing breaks."

## What We'd Do Differently

**Start with app-of-apps from day one.** We initially created Argo CD apps through the UI. By app #15 it was chaos. Migrating to app-of-apps retroactively meant exporting every `Application` as YAML, importing existing resources, and reconciling drift — a weekend that should have been avoided.

**Use ApplicationSets for repeated patterns.** We have four ApplicationSets for things deployed identically to every node (Alloy, kube-state-metrics, metrics-server, Prometheus CRDs). We should have more — anything that fans out over a list of clusters, namespaces, or teams is an ApplicationSet in waiting.

**Pin Helm chart versions aggressively.** We had an incident where an unpinned upstream chart auto-updated and broke Istio's mesh config. Now every `targetRevision` is an explicit tag or SHA — no `HEAD`, no `main`, no `*`.

**Keep values files small and environment-scoped.** The first pass put everything in one `values.yaml` per app. By month three the files were 400 lines of half-conditional templating. Splitting into `common-values.yaml` + `environments/<env>/<app>-values.yaml` made diffs readable again.

---

**Bare Metal K8s series:** [Part 1: Why](/blog/bare-metal-k8s-part1-why/) · [Part 2: Bootstrap](/blog/bare-metal-k8s-part2-bootstrap/) · **Part 3: GitOps** · [Part 4: Observability](/blog/bare-metal-k8s-part4-observability/) · [Part 5: AI Platform](/blog/bare-metal-k8s-part5-ai-platform/)

*Cloud Native Solutions builds and operates Kubernetes platforms end-to-end. [Talk to us](/contact/) if you want this for your team.*
