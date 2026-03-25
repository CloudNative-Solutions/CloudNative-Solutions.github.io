---
title: "Building an Enterprise Platform on Bare Metal — Part 2: Bootstrapping with Tinkerbell & Cluster API"
date: 2026-03-22
author: "Marius Oprin"
image: "/images/services/infrastructure-design.png"
tags: ["kubernetes", "bare-metal", "tinkerbell", "cluster-api", "series"]
draft: false
---

How do you provision bare-metal Kubernetes nodes without touching a USB stick? With Tinkerbell and Cluster API.

## The Problem with Manual Provisioning

Traditional bare-metal setup means: download ISO, flash USB, boot machine, run installer, configure networking, install container runtime, join cluster. For 3 nodes, that's annoying. For 30, it's a full-time job.

We wanted cloud-like provisioning: define a machine spec, hit apply, and the node boots itself into a working Kubernetes cluster member.

## Enter Tinkerbell

Tinkerbell is a CNCF project that provides bare-metal provisioning via PXE boot and workflows. Here's how it works:

1. **Machine powers on** and PXE boots from the network
2. **Tinkerbell boots a lightweight OS** image over the network
3. **Workflow engine runs provisioning steps**: partition disks, install OS, configure networking
4. **Machine reboots** into the installed OS, ready for Kubernetes

We use Flatcar Container Linux as our node OS — it's immutable, auto-updating, and purpose-built for containers.

## Cluster API: Declarative Cluster Management

Cluster API (CAPI) treats infrastructure like any other Kubernetes resource. You declare what you want:

```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: mgmt
spec:
  controlPlaneRef:
    kind: KubeadmControlPlane
    name: mgmt-control-plane
  infrastructureRef:
    kind: TinkerbellCluster
    name: mgmt
```

CAPI + Tinkerbell together give us fully automated bare-metal cluster lifecycle management. Need to upgrade Kubernetes? Change the version in the spec, apply, and CAPI handles rolling upgrades across all nodes.

## Our Setup

- **3 control plane nodes** — all untainted, serving as both control plane and workers
- **Cilium CNI** — eBPF-based networking, no kube-proxy
- **MetalLB** — load balancer for bare-metal (announces VIPs via ARP)
- **Kube-VIP** — HA control plane endpoint at 192.168.0.100

The entire cluster bootstrap is defined in our gitops repo under `bootstrap/` — Tinkerbell hardware specs, CAPI manifests, and Ansible playbooks for initial setup.

## Lessons Learned

1. **PXE boot is finicky** — BIOS settings matter. Disable Secure Boot, enable network boot, set boot order correctly.
2. **Network planning matters** — we use a dedicated VLAN for provisioning traffic.
3. **Start with 1 node** — bootstrap a single control plane node first, then use CAPI to add the rest.

---

*Next up: [Part 3](/blog/bare-metal-k8s-part3-gitops/) — How we manage 84 applications from a single GitOps repository.*
