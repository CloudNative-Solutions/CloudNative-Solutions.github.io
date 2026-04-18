---
title: "The Dynamic Duo of Kubernetes: Argo CD + Cluster API"
date: 2024-03-22
draft: false
author: "Marius Oprin"
image: "/images/blog/argocd-and-clusterapi/argocd-and-clusterapi.png"
tags: ["Argo CD", "Cluster API", "Kubernetes", "GitOps"]
description: "How Argo CD and Cluster API combine to make multi-cluster provisioning, configuration, and deployment fully declarative — with a real controller bridging the two and ApplicationSets doing the heavy lifting."
summary: "Argo CD knows how to deploy applications. Cluster API knows how to build clusters. Neither knows the other exists. Here is the controller that teaches them — and what the resulting pipeline looks like in practice."
---

Managing one Kubernetes cluster is manageable. Managing ten of them — with consistent add-ons, consistent policies, and zero manual steps — is where most teams start drowning. The combination of **Argo CD** and **Cluster API** fixes this, but only if you teach the two to talk to each other.

This post walks through the integration: the [argocd-capi-controller](https://github.com/CloudNative-Solutions/argocd-capi-controller) that bridges them, the `ApplicationSet` patterns that make it scale, and Argo CD's sync-wave feature that makes the order come out right.

## Why the Two Need a Bridge

**Argo CD** is a declarative, GitOps-style delivery tool for Kubernetes. You point it at a git repository and it reconciles the cluster state to match. Perfect for application deployment.

**Cluster API (CAPI)** is the opposite layer — it treats the cluster itself as a Kubernetes resource. You declare a `Cluster` plus some infrastructure refs and CAPI provisions the control plane, the worker nodes, and the networking. Perfect for cluster lifecycle.

Left to themselves, the two don't know the other exists. CAPI finishes provisioning a cluster and drops a kubeconfig in a secret; Argo CD has no idea the cluster is there. Someone still has to copy credentials, register the cluster in Argo CD, and point applications at it. That is the manual step everyone forgets and everyone does differently.

## The argocd-capi-controller

Our [argocd-capi-controller](https://github.com/CloudNative-Solutions/argocd-capi-controller) is a small Kubernetes controller that closes the loop:

1. **Watches CAPI** for `Cluster` objects reaching the `Provisioned` phase.
2. **Fetches the target kubeconfig** from the secret CAPI produces.
3. **Creates an Argo CD service account** in the target cluster, scoped to what Argo CD needs.
4. **Writes a cluster secret** in the Argo CD namespace with the label `argocd.argoproj.io/secret-type: cluster`. That label is the entire mechanism Argo CD uses to register new clusters.

No `argocd cluster add`. No copy-pasting kubeconfigs. A new cluster appears in Argo CD within seconds of CAPI finishing.

## Dynamic Provisioning with ApplicationSets

Once new clusters show up automatically, [ApplicationSets](https://argo-cd.readthedocs.io/en/stable/user-guide/application-set/) do the rest of the work. We use two generators, stacked.

**Step 1 — provision clusters from a git folder.** A [Git Generator](https://argocd-applicationset.readthedocs.io/en/stable/Generators-Git) watches a directory of CAPI values files. Every file in `clusters/` becomes a cluster:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: capi-clusters
  namespace: argocd
spec:
  generators:
    - git:
        repoURL: https://github.com/example/platform-gitops
        revision: main
        files:
          - path: "clusters/*/values.yaml"
  template:
    metadata:
      name: "{{ .path.basename }}-cluster"
    spec:
      project: default
      source:
        repoURL: https://github.com/example/platform-gitops
        path: charts/capt-chart
        targetRevision: main
        helm:
          valueFiles:
            - "/{{ .path.path }}"
      destination:
        server: https://kubernetes.default.svc
        namespace: capi-system
      syncPolicy:
        automated: { prune: true, selfHeal: true }
```

Add a new directory under `clusters/`, commit, push — Argo CD renders the Helm chart, CAPI provisions a cluster, and the controller above registers it. No UI clicks.

**Step 2 — install add-ons on every cluster.** A second `ApplicationSet`, this time using the [Cluster Generator](https://argocd-applicationset.readthedocs.io/en/stable/Generators-Cluster/), fans out over every registered cluster and installs the base stack — CNI, CSI, autoscaler, external-dns, load balancer controller:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: cluster-addons
  namespace: argocd
spec:
  generators:
    - clusters:
        selector:
          matchLabels:
            addons: enabled
  template:
    metadata:
      name: "{{ .name }}-addons"
    spec:
      project: default
      source:
        repoURL: https://github.com/example/platform-gitops
        path: "addons/{{ .metadata.labels.infra }}"
        targetRevision: main
      destination:
        server: "{{ .server }}"
        namespace: kube-system
      syncPolicy:
        automated: { prune: true, selfHeal: true }
```

The `clusters` generator picks up the secret that the argocd-capi-controller wrote. Label the cluster with `infra: aws` or `infra: tinkerbell` and the `ApplicationSet` deploys the right add-ons for that infrastructure.

## Ordering with Sync-Waves

Some things have to happen before others. CNI before anything that needs pod networking. Cert-manager before any `Ingress` that references a ClusterIssuer. External-DNS after the load balancer controller so it picks up the right external IP.

Argo CD's [sync-waves](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/) are how you express that ordering without writing a procedural script:

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "-2"  # CNI, first
---
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "0"   # Cert-manager, next
---
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "2"   # Ingress controllers, after certs
```

Lower waves apply first. Argo CD waits for each wave's resources to reach a healthy state before starting the next. One sync, correct order, every time.

## Why This Matters

The moment you have more than one cluster — dev, staging, production, a per-customer cluster — the manual overhead compounds. Teaching Argo CD and Cluster API to talk to each other collapses cluster provisioning, add-on installation, and application deployment into a single git-push workflow.

The pattern scales from one cluster to a fleet without changing shape. Add a file, get a cluster. Add a label, get an add-on. Add a sync-wave, get an order. That is the point.

## See It Running

We later applied this exact architecture on our own production platform — on bare metal instead of cloud, but the Argo CD + CAPI pipeline is identical. Worth reading together:

- [Bare Metal Kubernetes, Part 1 — Why we did it](/blog/bare-metal-k8s-part1-why/)
- [Part 2 — Bootstrapping with Tinkerbell and Cluster API](/blog/bare-metal-k8s-part2-bootstrap/)
- [Part 3 — One GitOps repo for 84 applications](/blog/bare-metal-k8s-part3-gitops/)

---

*Cloud Native Solutions builds and operates Kubernetes platforms end-to-end. [Talk to us](/contact/) if you want this for your team.*
