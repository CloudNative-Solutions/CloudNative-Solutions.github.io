---
title: "The Dynamic Duo of Kubernetes: Exploring Argo CD and Cluster API Integration"
date: 2024-03-22
draft: false
author: "Marius Oprin"
image: "/images/blog/argocd-and-clusterapi/argocd-and-clusterapi.png"
tags: ["Argo CD", "Cluster API", "Kubernetes"]
categories: ["GitOps"]
description: "Exploring the seamless integration of Argo CD and Cluster API for Kubernetes cluster management."
---

In the world of Kubernetes, managing multiple clusters and ensuring their configurations are as declared can be a challenging task. However, the integration of **Argo CD** with **Cluster API (CAPI)** is changing the game, making multi-cluster management seamless and more efficient. In this blog post, we'll delve into how Argo CD and Cluster API work together to automate the provisioning and configuration of Kubernetes clusters, focusing on the pivotal role of the [**argocd-capi-controller**](https://github.com/CloudNative-Solutions/argocd-capi-controller), the power of ApplicationSets, and the strategic deployment sequencing with Argo CD's sync-wave feature.

## The Synergy Between Argo CD and Cluster API

### Introduction to Argo CD and Cluster API

**Argo CD** is a declarative, GitOps continuous delivery tool for Kubernetes. It enables developers to define and control the deployment of applications and configurations in a Git repository, ensuring that the state of the clusters matches the declarations in Git.

**Cluster API (CAPI)**, on the other hand, is a Kubernetes sub-project focused on providing declarative APIs and tooling to simplify provisioning, upgrading, and operating multiple Kubernetes clusters. By treating cluster management as a Kubernetes-native application, CAPI abstracts away the complexities of interacting directly with infrastructure technologies.

### Unveiling the argocd-capi-controller

The [argocd-capi-controller](https://github.com/CloudNative-Solutions/argocd-capi-controller) acts as a bridge between Argo CD and CAPI, subscribing to the CAPI Kubernetes API to monitor newly created clusters. When these clusters reach a provisioned state, the controller performs several critical actions:
- Retrieves the target cluster's Kubeconfig.
- Creates an Argo CD service account in the target cluster.
- Within the Argo CD cluster, the controller creates a secret with the label `argocd.argoproj.io/secret-type: cluster`, which is essentially adding the target cluster to Argo CD. This secret contains all the connection and configuration details required by Argo CD.

This seamless integration not only automates the addition of clusters into Argo CD but also sets the stage for further configuration and management through ApplicationSets.

### Leveraging ApplicationSets for Dynamic Provisioning and Configuration

With the capability to automatically add clusters to Argo CD once they are provisioned by CAPI, we can now utilize [ApplicationSets](https://argo-cd.readthedocs.io/en/stable/user-guide/application-set/) to dynamically provision and configure our Kubernetes clusters.

1. **Initial Cluster Provisioning with Git Generator**: The first ApplicationSet uses a [Git Generator](https://argocd-applicationset.readthedocs.io/en/stable/Generators-Git), which reads CAPI values files from a specific folder structure in a Git repository to create Kubernetes clusters. This step automates the provisioning process based on predefined configurations, ensuring consistency and repeatability across cluster deployments.

2. **Configuring Clusters with Cluster Generator**: Once the clusters are provisioned and added to Argo CD, the next step involves the deployment of critical addons such as CSI (Container Storage Interface), CNI (Container Network Interface), autoscaler, external-dns, and load balancer controllers. For this, a [Cluster Generator](https://argocd-applicationset.readthedocs.io/en/stable/Generators-Cluster/) within an ApplicationSet is used. This generator dynamically selects the newly added clusters and deploys the necessary addons, preparing the target clusters to host various applications.

### Optimizing Deployment Order with Argo CD's Sync-Wave Feature

To ensure a seamless and error-free rollout of applications across these clusters, we employ Argo CD's [sync-wave feature](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/). This feature enables us to specify the order of resource deployment within the same application, ensuring that dependencies are correctly resolved. By controlling the sequencing of deployment, we can ensure that Argo CD applications are deployed in the right order, making sure that dependencies are satisfied.

## Conclusion

The integration of Argo CD with Cluster API through the [argocd-capi-controller](https://github.com/CloudNative-Solutions/argocd-capi-controller) and the strategic use of ApplicationSets presents a robust solution for managing Kubernetes clusters at scale. By leveraging the sync-wave feature for deployment order, this synergy not only automates the tedious tasks of cluster provisioning and configuration but also ensures that applications are deployed in a precise, dependency-aware manner. With Argo CD and Cluster API working in harmony, the process of deploying and managing applications across multiple clusters is not just automated but refined to meet the nuanced demands of modern cloud-native environments.

## To Be Continued

Next, we'll demonstrate the automatic deployment of AWS EKS clusters using Argo CD and Cluster API. Stay tuned for a practical guide on implementing these technologies in action.
