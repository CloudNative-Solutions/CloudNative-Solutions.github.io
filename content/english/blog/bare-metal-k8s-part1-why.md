---
title: "Building an Enterprise Platform on Bare Metal — Part 1: Why We Did It"
date: 2026-03-20
author: "Marius Oprin"
image: "/images/case-studies/bare-metal-platform.png"
tags: ["kubernetes", "bare-metal", "infrastructure", "series"]
draft: false
---

Everyone told us to use the cloud. We chose Intel NUCs instead.

Not because we're contrarian — because the economics made sense. Running a full enterprise platform on 3 mini-PCs costs roughly €1,500 one-time versus €2,000+/month on any major cloud provider. For a small DevOps consultancy, that difference is the margin between profitable and burning cash.

## The Cloud Tax Problem

We needed to run our entire platform: CI/CD pipelines, container registry, monitoring stack, identity management, service mesh, and an AI agent platform. On AWS, that's easily 10+ services, each with its own pricing model, each designed to make you forget how much you're spending.

Quick napkin math for our workload on AWS:
- 3x m6i.xlarge instances: ~$450/mo
- EKS control plane: $73/mo
- EBS storage (1.5TB): ~$90/mo
- ALB + data transfer: ~$150/mo
- RDS for Keycloak + apps: ~$200/mo
- S3 for backups + artifacts: ~$50/mo
- **Total: ~$1,000/mo minimum** — and that's before any AI/ML GPU workloads

Our 3 NUCs? One-time cost. We pay for electricity and internet. That's it.

## What We Needed to Run

This isn't a hobby cluster. We needed production-grade infrastructure:

- **84 applications** across DevOps, observability, security, AI/ML, and collaboration
- **GitOps-managed** — every change tracked, auditable, reproducible
- **SSO everywhere** — Keycloak integrated with 15+ services
- **Full observability** — metrics, logs, traces, profiling, error tracking
- **Security** — runtime protection, policy enforcement, secrets management
- **AI platform** — LLM gateway, agent orchestration, experiment tracking
- **VM workloads** — KubeVirt for running traditional VMs alongside containers, no separate hypervisor needed

The question was never "can Kubernetes run on bare metal?" — it was "can we build something that rivals cloud-managed platforms, on hardware we own?"

Spoiler: yes.

## The Hardware

3x Intel NUC 13 Pro:
- Intel Core i5-1350P (12 cores, 16 threads)
- 64GB DDR4 RAM each
- 500GB NVMe SSD each
- 2.5GbE networking

Total: 36 cores, 192GB RAM, 1.5TB NVMe storage. Rack-mounted in a home lab, connected to a UPS.

## What's Coming in This Series

- **Part 2:** Bootstrapping — Tinkerbell, Cluster API, and automated provisioning
- **Part 3:** The GitOps architecture — 84 apps, one repo, zero manual deployments
- **Part 4:** Observability at scale — LGTM stack with multi-tenant Mimir
- **Part 5:** Running an AI agent platform on Kubernetes

Each post will include real configs, lessons learned, and things we'd do differently.

---

*Cloud Native Solutions builds and operates Kubernetes platforms. [Talk to us](/contact/) if you want this for your team.*
