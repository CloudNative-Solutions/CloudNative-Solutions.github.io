---
title: "Building an Enterprise Platform on Bare Metal — Part 3: GitOps for 84 Applications"
date: 2026-03-24
author: "Marius Oprin"
image: "/images/services/platform-engineering.png"
tags: ["kubernetes", "gitops", "argocd", "kargo", "series"]
draft: false
---

One git repository. 84 applications. Zero manual deployments. Here's how we structured our GitOps architecture.

## The App-of-Apps Pattern

ArgoCD's app-of-apps pattern is how we manage scale. Instead of creating 84 individual ArgoCD Application resources manually, we organize apps into categories:

```
mgmt-cluster/
  apps/
    agentic/          # 17 AI/ML apps
    devops/           # 12 CI/CD apps  
    observability/    # 12 monitoring apps
    security/         # 7 security apps
    identity/         # 4 IAM apps
    infrastructure/   # 8 infra apps
    networking/       # 3 network apps
    storage/          # 2 storage apps
    service-mesh/     # 4 mesh apps
    collaboration/    # 3 collab apps
    ...
```

Each category has a Kustomization that references its apps. A root Application points to all categories. Add a new app? Create a YAML file, commit, push — ArgoCD picks it up automatically.

## Self-Healing with ArgoCD

Every application has `selfHeal: true` and `prune: true`:

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

Someone `kubectl edit`s a deployment directly? ArgoCD reverts it within seconds. This is how we ensure the git repo is always the source of truth. No drift, no surprises.

## Environment Promotion with Kargo

Kargo handles promotion pipelines across environments. When a new Helm chart version is pushed to our registry, Kargo:

1. Detects the new version
2. Updates the values file in the gitops repo
3. Creates a PR (or auto-promotes for non-production)
4. ArgoCD syncs the change

This gives us a clear audit trail: who promoted what, when, and why.

## Custom Helm Charts

We maintain 11 custom Helm charts in our `catalog/`:
- Platform applications (Mission Control, DevOps AI, O8s Cloner)
- Infrastructure utilities (Postfix relay, common templates)
- Each chart follows a consistent structure with sensible defaults

## The Golden Rule

**Never `kubectl apply` in production.** Every change goes through git. Even "quick fixes." Especially quick fixes.

We learned this the hard way when ArgoCD's self-heal reverted a manual `kubectl set resources` change within seconds. That was the day we committed to pure GitOps — and it's been worth every extra minute spent on PRs.

---

*Next up: Part 4 — Observability at scale with the LGTM stack.*
