---
title: "Enterprise Platform on Bare Metal: 84 Apps, 3 Nodes, Zero Cloud Bills"
meta_title: "Case Studies"
description: "How we built an enterprise-grade Kubernetes platform on 3 Intel NUCs with full CNCF stack"
image: /images/case-studies/bare-metal-platform.png
draft: false
weight: 1
---

## The Challenge

Build a production-grade platform that rivals enterprise cloud deployments — on three Intel NUC mini-PCs. No cloud provider, no managed services, no monthly bills. Full GitOps, full observability, full security.

## The Solution

We deployed a bare-metal Kubernetes cluster using Cluster API with Tinkerbell for machine provisioning. The entire platform is managed through a single GitOps repository with ArgoCD, spanning 84 applications across 15 categories.

### Infrastructure Layer
- **Cluster Topology:** 3 control plane nodes running untainted — serving as both control plane and workers for maximum resource utilization
- **Bare Metal Provisioning:** Cluster API + Tinkerbell for automated node provisioning
- **Networking:** Cilium CNI, MetalLB for load balancing, Istio service mesh
- **Storage:** Rook-Ceph distributed storage across all three nodes (1.5TB NVMe, 42% utilized)
- **DNS & TLS:** External-DNS, Cert-Manager with Let's Encrypt, Traefik ingress

### Platform Layer
- **GitOps:** ArgoCD with app-of-apps pattern, self-healing enabled
- **CI/CD:** Argo Workflows + Argo Events for event-driven pipelines
- **Progressive Delivery:** Argo Rollouts, Kargo for environment promotion
- **Developer Portal:** Backstage for service catalog and golden paths
- **Artifact Registry:** Gitea (Git) + Harbor (container images)

### Security Layer
- **Identity:** Keycloak SSO integrated with 15+ services via OIDC
- **Secrets:** HashiCorp Vault + External Secrets Operator
- **Runtime:** Falco threat detection, Kyverno policy enforcement
- **Supply Chain:** Trivy scanning, Harbor vulnerability checks

### Observability Layer
- **Metrics:** Mimir for scalable, multi-tenant metrics storage (Prometheus-compatible) + Grafana
- **Logs:** Loki with S3-backed storage
- **Traces:** Tempo for distributed tracing
- **Telemetry Collection:** Grafana Alloy as the unified collector for metrics, logs, and traces
- **Profiling:** Pyroscope for continuous profiling
- **Error Tracking:** Sentry with ClickHouse backend
- **Service Mesh:** Kiali for Istio observability

### AI/ML Platform
- **LLM Gateway:** LiteLLM routing to multiple AI providers
- **Observability:** Langfuse for LLM tracing and evaluation
- **Chat Interface:** Open WebUI for model interaction
- **Local Inference:** NVIDIA DGX Spark running Qwen models — zero-latency, zero-cost local LLM serving
- **Agent Platform:** 18 AI agents with orchestration layer
- **Automation:** n8n workflows, NATS messaging

### Collaboration
- **Chat:** Synapse (Matrix) for team communication
- **Files:** Nextcloud for document management
- **Automation:** n8n for workflow automation

## The Results

- **84 applications** deployed and managed via GitOps
- **Zero cloud bills** — entire platform on 3x Intel NUC hardware
- **99.9% uptime** with automated self-healing (ArgoCD, VPA, kured)
- **30+ Ceph-backed persistent volumes** across workloads
- **15+ services** with SSO via Keycloak
- **Full CNCF stack** — no vendor lock-in
- **Local AI inference** via NVIDIA DGX Spark — zero API costs for development and testing
- **Single git repo** manages everything — infrastructure to applications
