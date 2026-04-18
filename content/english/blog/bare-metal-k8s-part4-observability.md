---
title: "Building an Enterprise Platform on Bare Metal — Part 4: Observability with the LGTM Stack"
date: 2026-03-26
author: "Marius Oprin"
image: "/images/services/monitoring-and-logs.png"
tags: ["kubernetes", "observability", "grafana", "mimir", "loki", "tempo", "series"]
draft: false
description: "Metrics, logs, traces, and continuous profiles for a whole production platform — running on the same three bare-metal nodes, with a single telemetry collector and multi-tenant storage."
summary: "You can't manage what you can't see. Here is how we run the full LGTM stack on the same three NUCs as the workloads it observes — multi-tenant, S3-backed on Ceph, with Grafana Alloy as the only agent on every node."
---

You can't manage what you can't see. Our observability stack handles metrics, logs, traces, and continuous profiles for every workload on the cluster — multi-tenant, all on the same three NUCs as the things it observes, all S3-backed against an internal Ceph cluster.

## The LGTM Stack

LGTM is Grafana's open-source observability stack:

- **Mimir** — long-term metrics storage, Prometheus-compatible, horizontally scalable, natively multi-tenant. Replaces a single-instance Prometheus with one that can actually grow.
- **Loki** — log aggregation. Prometheus-style label-based querying, no full-text index, dramatically cheaper than Elasticsearch for the same retention.
- **Tempo** — distributed tracing backend. Accepts OTLP, Jaeger, and Zipkin. Stores traces on object storage with a hashed span-id index.
- **Grafana** — the front end. Single pane of glass for everything.

All four speak to the same S3-compatible bucket, served by Rook-Ceph on the cluster itself. No external S3, no cross-region egress, no surprise AWS bill.

## Grafana Alloy: One Agent on Every Node

Alloy is Grafana's unified telemetry collector — the successor to the Grafana Agent and a drop-in replacement for separate node-exporters, Promtail, and OTel collectors. One DaemonSet, one config, one thing to upgrade:

```river
prometheus.scrape "kubelet" {
  targets    = discovery.kubernetes.nodes.targets
  forward_to = [prometheus.remote_write.mimir.receiver]
  scrape_interval = "30s"
}

loki.source.kubernetes "pods" {
  targets    = discovery.kubernetes.pods.targets
  forward_to = [loki.write.loki.receiver]
}

otelcol.receiver.otlp "default" {
  grpc { endpoint = "0.0.0.0:4317" }
  http { endpoint = "0.0.0.0:4318" }
  output { traces = [otelcol.exporter.otlp.tempo.input] }
}
```

Alloy runs as an `ApplicationSet`-managed DaemonSet — as soon as a new node joins the cluster, Alloy is on it, collecting, forwarding. No manual step.

## Multi-Tenancy, in Practice

Mimir and Loki are configured for multi-tenancy from day one. Every team, every environment, and every noisy subsystem gets its own tenant ID, with isolated ingestion, storage, and query paths.

The tenant list lives in a single ConfigMap:

```yaml
overrides:
  platform:
    ingestion_rate: 100000
    max_global_series_per_user: 5000000
    compactor_blocks_retention_period: 90d
  agents:
    ingestion_rate: 50000
    compactor_blocks_retention_period: 14d
  dev:
    ingestion_rate: 20000
    compactor_blocks_retention_period: 7d
```

What this buys us:

- **No noisy-neighbour problems** — a chatty agent can't starve platform metrics out of ingestion.
- **Per-tenant retention** — we keep platform metrics for 90 days, agent telemetry for two weeks, dev for a week. Storage bill stays flat.
- **Tenant-scoped dashboards** — Grafana data sources are pre-scoped to a tenant, so a dev dashboard can never accidentally query the whole cluster.

Loki's tenant split mirrors Mimir's exactly. We route by the `X-Scope-OrgID` header Alloy attaches based on a pod label.

## Beyond Metrics and Logs

Three more pieces complete the picture:

**Pyroscope** — continuous CPU and memory profiling across the cluster. Every Go binary we build exports `net/http/pprof`; Alloy scrapes it the same way it scrapes Prometheus. When a service gets slow, we don't try to reproduce — we go to Pyroscope and look at the flame graph from the incident window.

**Sentry** — error tracking with a ClickHouse backend. Every frontend, backend, and worker sends stack traces, breadcrumbs, and release markers. Integrated with Kargo so a new deploy appears as a release in Sentry automatically.

**k8sgpt** — AI-powered cluster diagnostics. Runs on a CronJob every ten minutes, scans the cluster for unhealthy objects, and posts plain-English explanations to Slack. It has caught misconfigured `NetworkPolicies`, failing readiness probes, and a PVC that had been pending for three hours before anyone looked at the namespace.

## Storage: Everything on Ceph

Every byte of observability data lands on Rook-Ceph object storage running on the same three nodes:

- **Mimir blocks** — compacted every two hours, uploaded to bucket `mimir-blocks`.
- **Loki chunks** — flushed every five minutes, uploaded to bucket `loki-chunks`.
- **Tempo blocks** — uploaded every 30 seconds for recent traces, compacted hourly.

Current storage sits at **~250 GB** combined across all three backends, with the retention configuration above. Ceph's replication factor of 3 means we pay for that capacity three times — which is fine at these volumes and the price of not having a SAN.

## What We'd Do Differently

**Turn on tenancy from day one.** We ran single-tenant for the first three months "to keep things simple" and then had to migrate. Migration is annoying — new bucket paths, new data-source config in Grafana, backfill gymnastics. Start multi-tenant; you can collapse tenants later if you really need to.

**Budget for Ceph.** Replicated object storage is not free. Sizing the cluster for "metrics, logs, traces × 3 replicas × 90 days" is a real number; do it before you tell anyone the retention policy.

**Alerts on the collector, not just the backends.** A dead Alloy pod is a silent observability outage. We alert on Alloy's own heartbeat series — if Alloy on node X stops reporting, we hear about it before the dashboards go dark.

---

**Bare Metal K8s series:** [Part 1: Why](/blog/bare-metal-k8s-part1-why/) · [Part 2: Bootstrap](/blog/bare-metal-k8s-part2-bootstrap/) · [Part 3: GitOps](/blog/bare-metal-k8s-part3-gitops/) · **Part 4: Observability** · [Part 5: AI Platform](/blog/bare-metal-k8s-part5-ai-platform/)

*Cloud Native Solutions builds and operates Kubernetes platforms end-to-end. [Talk to us](/contact/) if you want this for your team.*
