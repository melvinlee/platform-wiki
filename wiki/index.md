---
name: index
description: Global wiki index — one entry per article, grouped by topic.
type: index
---

# Knowledge Base Index

## grafana-loki

How Grafana Loki ingests, indexes, and isolates log data.

| Article | Summary | Updated |
|---------|---------|---------|
| [Labels in Loki](grafana-loki/labels.md) | How Loki uses labels to define log streams, what makes a good label, and how streams are created. | 2026-04-15 |
| [Cardinality in Loki](grafana-loki/cardinality.md) | Why high label cardinality breaks Loki, how to detect it, and how to fix it with structured metadata and label discipline. | 2026-04-15 |
| [Structured Metadata in Loki](grafana-loki/structured-metadata.md) | What structured metadata is, when to use it instead of labels, how to enable and emit it, and how to query it (including the UI quirks). | 2026-04-15 |
| [Multi-tenancy in Loki](grafana-loki/multi-tenancy.md) | How Loki isolates tenant data with X-Scope-OrgID, and how to wire up auth, reverse proxies, Promtail pipelines, and Grafana datasources. | 2026-04-15 |

## anomaly-detection

AI/ML-based anomaly detection for observability and time-series data.

| Article | Summary | Updated |
|---------|---------|---------|
| [Anomaly Detection Overview](anomaly-detection/anomaly-detection-overview.md) | Types of anomalies, why detection matters for observability, best practices, commercial platform capabilities, and emerging trends. | 2026-04-16 |
| [Prophet for Time-Series Anomaly Detection](anomaly-detection/prophet-anomaly-detection.md) | How to use Meta's Prophet library for time-series anomaly detection in Python, with two approaches and model evaluation. | 2026-04-16 |

## llm-observability

Using LLMs for log analysis, anomaly detection, and automated root cause analysis in observability workflows.

| Article | Summary | Updated |
|---------|---------|---------|
| [LLM-based Log Analysis](llm-observability/llm-log-analysis.md) | Using LLMs to parse, structure, summarize, and detect anomalies in log files — techniques, hybrid pipelines, and guardrails. | 2026-04-16 |
| [Automating RCA with LLMs and MCP](llm-observability/llm-automated-rca.md) | Architecture for automated RCA using Golden Signals alerts, LLM agents, and MCP tools to fetch data and generate root-cause analysis. | 2026-04-16 |
