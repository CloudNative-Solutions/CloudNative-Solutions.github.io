---
title: "Building an Enterprise Platform on Bare Metal — Part 2: Bootstrapping with Tinkerbell & Cluster API"
date: 2026-03-22
author: "Marius Oprin"
image: "/images/services/infrastructure-design.svg"
tags: ["kubernetes", "bare-metal", "tinkerbell", "cluster-api", "series"]
draft: false
description: "How we provision bare-metal Kubernetes nodes with zero manual steps — using Tinkerbell for PXE boot and Cluster API for declarative cluster lifecycle."
summary: "How do you provision bare-metal Kubernetes without touching a USB stick? With Tinkerbell for PXE-based OS install and Cluster API turning the whole thing into Kubernetes resources you git commit."
---

How do you provision bare-metal Kubernetes nodes without touching a USB stick? With Tinkerbell and Cluster API — and a little discipline about what lives in git.

## The Problem with Manual Provisioning

Traditional bare-metal setup means: download ISO, flash USB, boot machine, run installer, configure networking, install container runtime, join cluster. For three nodes that is merely annoying. For thirty, it is a full-time job and an endless source of drift.

We wanted cloud-like provisioning: declare a machine spec, hit apply, and the node boots itself into a working Kubernetes cluster member. The same workflow for node #1, node #30, and any replacement after a disk failure.

## How Tinkerbell Works

Tinkerbell is a CNCF project for bare-metal provisioning over the network. The flow for one machine:

1. **Machine powers on** and requests a network boot (PXE).
2. **Tinkerbell's boot server** serves a lightweight in-memory OS (HookOS).
3. **The workflow engine** runs a templated sequence of actions: partition disks, stream the OS image to NVMe, write an Ignition config, reboot.
4. **Machine boots from disk** into Flatcar Container Linux, picks up its Ignition config, and joins the cluster.

We use Flatcar as the node OS: immutable root filesystem, automated updates (A/B partitions), container runtime pre-installed, ignition-driven configuration.

A Tinkerbell `Template` for a Flatcar install looks like this:

```yaml
apiVersion: tinkerbell.org/v1alpha1
kind: Template
metadata:
  name: flatcar-install
  namespace: tink-system
spec:
  data: |
    version: "0.1"
    name: flatcar-install
    global_timeout: 1800
    tasks:
      - name: os-install
        worker: "{{.device_1}}"
        actions:
          - name: stream-image
            image: quay.io/tinkerbell-actions/image2disk:v1.0.0
            environment:
              IMG_URL: http://192.168.0.200:8080/flatcar_production_image.bin.bzip2
              DEST_DISK: /dev/nvme0n1
              COMPRESSED: true
          - name: write-ignition
            image: quay.io/tinkerbell-actions/writefile:v1.0.0
            environment:
              DEST_DISK: /dev/nvme0n1
              FS_TYPE: ext4
              DEST_PATH: /ignition.json
              CONTENTS: "{{.ignition_config}}"
              MODE: "0644"
          - name: reboot
            image: ghcr.io/jacobweinstock/waitdaemon:latest
            timeout: 90
            command: ["reboot"]
```

Each physical machine has a `Hardware` record with its MAC address, the network interface Tinkerbell speaks to, and any labels (manufacturer, role, CPU topology) that Cluster API will match against later.

## Cluster API: Declarative Cluster Lifecycle

Tinkerbell handles the OS on a single box. Cluster API (CAPI) handles what happens next: bootstrapping kubeadm, joining control planes, rolling upgrades, replacing unhealthy nodes. All of it expressed as Kubernetes resources.

Here is the top of our management-cluster definition — the `Cluster` plus its Tinkerbell infrastructure reference:

```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: mgmt
  namespace: tink-system
spec:
  clusterNetwork:
    pods:
      cidrBlocks: ["192.167.0.0/16"]
    services:
      cidrBlocks: ["172.26.0.0/16"]
  controlPlaneEndpoint:
    host: 192.168.0.100
    port: 6443
  controlPlaneRef:
    apiVersion: controlplane.cluster.x-k8s.io/v1beta1
    kind: KubeadmControlPlane
    name: mgmt-control-plane
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
    kind: TinkerbellCluster
    name: mgmt
---
apiVersion: controlplane.cluster.x-k8s.io/v1beta1
kind: KubeadmControlPlane
metadata:
  name: mgmt-control-plane
  namespace: tink-system
spec:
  replicas: 3
  version: v1.30.0
  machineTemplate:
    infrastructureRef:
      apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
      kind: TinkerbellMachineTemplate
      name: mgmt-control-plane
```

Need to upgrade Kubernetes? Bump the `version` field, commit, push — CAPI does a rolling replacement across all nodes. Need a fourth control plane? Change `replicas` from 3 to 4 and Tinkerbell provisions the new hardware on the next PXE boot.

The entire cluster bootstrap lives in our GitOps repo under `bootstrap/`: the Tinkerbell Hardware records, the CAPI manifests, the Flatcar Ignition templates, and the Kube-VIP manifest we inject for HA control-plane endpoint.

## Our Setup

- **Three control plane nodes, untainted** — they serve as control plane and workers. 36 cores across the cluster is not enough to dedicate any of it to an idle control plane.
- **Cilium CNI** — eBPF-based, no kube-proxy. Drops ~1% of per-request CPU at scale.
- **MetalLB** — layer-2 announcements for load-balancer services. ARP is enough on a single VLAN.
- **Kube-VIP** — floats `192.168.0.100:6443` across the three control plane nodes for HA.
- **Dedicated provisioning VLAN** — PXE and DHCP traffic stay isolated from the production network.

## Lessons Learned

**PXE boot is finicky — budget extra time for BIOS.** Secure Boot disabled, network boot enabled, boot order set correctly on every node. NUC BIOS defaults work about 70% of the time; the other 30% you find yourself on a KVM fixing one setting.

**Plan the network before you plan the workflows.** A dedicated provisioning VLAN keeps DHCP from leaking into production. MetalLB uses a separate IP range from anything else on the network; kube-vip uses its own. Write it all down before you start.

**Bootstrap a single node manually, then let CAPI take over.** You need one working control-plane node to run the CAPI operators. Once CAPI is up, it provisions the rest. Trying to bootstrap all three in one shot is how you spend a Saturday debugging Ignition.

**Pin every image URL.** Flatcar channel URLs move; Tinkerbell action images publish new tags. We pin SHAs for actions and point at a local HTTP server for the Flatcar image — no surprise updates mid-provision.

---

**Bare Metal K8s series:** [Part 1: Why](/blog/bare-metal-k8s-part1-why/) · **Part 2: Bootstrap** · [Part 3: GitOps](/blog/bare-metal-k8s-part3-gitops/) · [Part 4: Observability](/blog/bare-metal-k8s-part4-observability/) · [Part 5: AI Platform](/blog/bare-metal-k8s-part5-ai-platform/)

*Cloud Native Solutions builds and operates Kubernetes platforms end-to-end. [Talk to us](/contact/) if you want this for your team.*
