---
title: "Building an Enterprise Platform on Bare Metal — Part 4: Observability with the LGTM Stack"
date: 2026-03-26
author: "Marius Oprin"
image: "/images/services/monitoring-and-logs.png"
tags: ["kubernetes", "observability", "grafana", "mimir", "loki", "tempo", "series"]
draft: false
---

You can't manage what you can't see. Our observability stack handles metrics, logs, traces, and profiling — all multi-tenant, all on the same 3 NUCs.

## The LGTM Stack

LGTM stands for Loki, Grafana, Tempo, Mimir — Grafana's open source observability stack:

- **Mimir** — Long-term metrics storage, Prometheus-compatible. Replaces standalone Prometheus with a scalable, multi-tenant backend.
- **Loki** — Log aggregation. Like Prometheus, but for logs. Label-based querying without indexing full log content.
- **Tempo** — Distributed tracing backend. Stores traces from OpenTelemetry, Jaeger, or Zipkin.
- **Grafana** — Visualization. Single pane of glass for everything.

## Grafana Alloy: The Universal Collector

Alloy is Grafana's unified telemetry collector. It replaces the need for separate Prometheus node-exporters, log scrapers, and trace collectors. One agent collects everything:

- Metrics from Kubernetes API, nodes, and applications
- Logs from container stdout/stderr and system journals
- Traces via OpenTelemetry protocol
- Profiles via Pyroscope integration

We deploy Alloy as a DaemonSet via ApplicationSet — it automatically runs on every node.

## Multi-Tenancy

Mimir and Loki are configured for multi-tenancy. Each team or application gets its own tenant ID, with isolated storage and query paths. This means:

- No noisy neighbor problems
- Per-tenant retention policies
- Tenant-scoped dashboards in Grafana

## Beyond the Basics

We also run:
- **Pyroscope** — Continuous profiling. Find CPU and memory hotspots without reproducing issues.
- **Sentry** — Error tracking with ClickHouse backend. Stack traces, breadcrumbs, and release tracking.
- **k8sgpt** — AI-powered cluster diagnostics. Points out issues before they become incidents.

## Storage Considerations

All observability data is backed by Ceph object storage (via Rook). Mimir, Loki, and Tempo write to S3-compatible buckets on the same Ceph cluster that serves our persistent volumes. No external S3 needed.

Current usage: ~250GB for metrics, logs, and traces combined, with configurable retention.

---

*Next up: Part 5 — Running an AI agent platform on Kubernetes.*
