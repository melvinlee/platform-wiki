---
title: Cardinality in Loki
summary: Why high label cardinality breaks Loki, how to detect it, and how to fix it with structured metadata and label discipline.
updated: 2026-04-15
sources: Grafana Labs, Loki docs (undated); Nawaz Dhandala (OneUptime), 2026-01-21; jose (Canonical / Charmhub), 2024-09-06; Rudi Martinsen, 2024-12-18
raw: [grafana-cardinality-docs](../../raw/grafana-loki/grafana-cardinality-docs.md); [how-to-optimize-loki-label-cardinality](../../raw/grafana-loki/2026-01-21-how-to-optimize-loki-label-cardinality.md); [solving-high-cardinality-labels-in-loki](../../raw/grafana-loki/2024-09-06-solving-high-cardinality-labels-in-loki.md); [working-with-structured-metadata-in-grafana-loki](../../raw/grafana-loki/2024-12-18-working-with-structured-metadata-in-grafana-loki.md)
---

# Cardinality in Loki

**Cardinality** in Loki refers to the combination of label keys and values and the number of log streams they create. Because a stream is defined by its full label set, each unique combination of label values is a new stream, a new index entry, and a new chunk in object storage.

Loki is built for **long-lived, low-cardinality streams**. High cardinality is the single most common cause of Loki performance and cost problems.

> Contrast with SQL: in SQL you *want* high-cardinality indexes because they narrow queries quickly. In Loki, high cardinality does the opposite — it multiplies streams, inflates the index, and flushes thousands of tiny chunks.

## Why high cardinality breaks Loki

Stream count = **product** of the cardinalities of all index labels.

- `job(5) × instance(100) × level(4)` = 2,000 streams — fine.
- `job(5) × user_id(100,000)` = 500,000 streams — broken.

High cardinality means:

- Larger index (more entries)
- More, smaller chunks flushed to object storage (more S3/GCS API calls)
- Slower queries (more series to scan)
- Higher ingester memory
- Ingestion failures once stream limits hit

The compounding effect is subtle: even low-cardinality labels multiply. Combining `status_code` (3 values) and `action` (5 values) is 15 streams. Adding an `endpoint` label with 3 values makes it 45.

## What causes it

Labels to **avoid**:

| Label | Why |
|-------|-----|
| `request_id`, `trace_id` | unique per request — unbounded |
| `user_id`, `customer_id` | unbounded |
| `session_id` | short-lived, unique |
| `timestamp`, `log_time` | already indexed |
| IP addresses | many unique values |
| Full paths, URLs | many variations |
| `filename` (in per-VM setups) | one per VM — grows with fleet |
| Kubernetes pod names | churn on every rollout |
| `process_id`, `thread_id` | ephemeral |

A concrete anti-example: OpenStack scraped with a `filename` label produces one stream per VM log file. A build-farm cloud spinning up thousands of VMs creates thousands of new streams per day.

Labels that are **appropriate** (bounded, stable):

| Label | Example |
|-------|---------|
| Environment | `env=prod` |
| Service | `service=api` |
| Log level | `level=error` |
| Namespace / team | `namespace=payments` |
| Region / cluster | `region=us-east-1` |
| Node (if bounded) | `node=worker-3` |

## Detection

**Via logcli:**

```bash
logcli series '{}' --since=1h --analyze-labels
```

**Via Prometheus metrics:**

```promql
# total active streams
sum(loki_ingester_memory_streams)

# per tenant
sum by (tenant) (loki_ingester_memory_streams)

# stream creation rate — a spike = cardinality explosion
rate(loki_ingester_streams_created_total[5m])
```

**Via the Loki API:**

```bash
curl -s 'http://loki:3100/loki/api/v1/label/__name__/values' | jq

# per-label cardinality
for label in $(curl -s 'http://loki:3100/loki/api/v1/labels' | jq -r '.data[]'); do
  count=$(curl -s "http://loki:3100/loki/api/v1/label/$label/values" | jq '.data | length')
  echo "$label: $count"
done
```

**Warning signs in Loki logs:** messages containing `cardinality`, `max streams`, or rate-limit errors.

## Remediation

### 1. Move high-cardinality fields out of labels

The simplest fix is: don't make it a label. Keep it in the log line and filter at query time.

```yaml
# Promtail — DO NOT do this
pipeline_stages:
  - json:
      expressions:
        user_id: user_id
  - labels:
      user_id:   # millions of streams
```

```yaml
# Promtail — correct
pipeline_stages:
  - json:
      expressions:
        user_id: user_id
  # no labels: user_id — filter at query time instead
```

Query with:

```logql
{job="app"} | json | user_id="12345"
```

### 2. Use structured metadata

Structured metadata (Loki 2.7+; GA in 2.9.4) stores per-entry metadata that is attached to log entries **without being indexed as labels and without creating new streams**. It's the intended solution for trace IDs, request IDs, pod names, filenames, and similar fields you still want to filter on.

Minimum requirements: `allow_structured_metadata: true` in `limits_config`, and schema v13 or newer in `schema_config`. The pipeline stage on the collector is `structured_metadata:` — use `labeldrop:` after it if the field was already extracted as a label.

Query with `|`, not `{}`:

```logql
{job="varlog_scraper"} | filename="/var/log/pepe.log"
```

`{job="varlog_scraper", filename="/var/log/pepe.log"}` returns empty — structured metadata is not a stream-selector label.

See [structured-metadata](structured-metadata.md) for full configuration, emission patterns, and Grafana quirks.

### 3. Bucket continuous values

For numeric fields where you want rough categorization but not per-value streams:

```yaml
pipeline_stages:
  - json:
      expressions:
        duration_ms: duration
  - template:
      source: duration_bucket
      template: '{{ if lt .duration_ms 100 }}fast{{ else if lt .duration_ms 1000 }}medium{{ else }}slow{{ end }}'
  - labels:
      duration_bucket:   # 3 values instead of millions
```

### 4. Drop labels at the collector

```yaml
# Promtail — labelkeep / labeldrop in relabel_configs
relabel_configs:
  - action: labelkeep
    regex: (job|namespace|pod|container|node)
  - action: labeldrop
    regex: (pod_uid|controller_revision_hash|pod_template_hash)
```

## Configure limits

Cardinality limits act as a safety net when a misconfigured client tries to create streams explosively.

```yaml
limits_config:
  max_streams_per_user: 10000
  max_global_streams_per_user: 50000
  max_label_names_per_series: 15
  max_label_name_length: 1024
  max_label_value_length: 2048
  max_labels_size_bytes: 4096
```

Per-tenant overrides:

```yaml
overrides:
  high-volume-tenant:
    max_streams_per_user: 5000
    max_label_names_per_series: 10
  platform:
    max_streams_per_user: 50000
    max_label_names_per_series: 20
```

## Monitoring

Alert on stream count, creation rate, and dropped samples:

```yaml
groups:
  - name: loki-cardinality
    rules:
      - alert: LokiHighStreamCount
        expr: sum by (tenant) (loki_ingester_memory_streams) > 50000
        for: 10m
      - alert: LokiStreamCreationSpike
        expr: rate(loki_ingester_streams_created_total[5m]) > 100
        for: 5m
      - alert: LokiMaxStreamsReached
        expr: loki_discarded_samples_total{reason="per_user_series_limit"} > 0
        for: 1m
```

## Migration playbook

When an existing deployment already has cardinality issues:

1. **Identify** the offending labels via the detection queries above.
2. **Update the collector** (Promtail / Alloy / Grafana Agent) to stop emitting them as labels — either drop them, move to structured metadata, or leave in content.
3. **Update queries and dashboards** to filter the content/metadata form instead of labels.
4. **Wait** for retention to expire old high-cardinality streams — they won't disappear immediately.
5. **Monitor** `loki_ingester_memory_streams` for the decline.

## Best practices summary

1. Ask "is this value bounded?" before adding any label.
2. Keep label count ≤ 10–15.
3. Use structured metadata for high-cardinality fields you still need to query.
4. Set `max_streams_per_user` as a safety net.
5. Alert on stream count and creation rate.
6. Audit label usage periodically; document team standards.

## See also

- [labels](labels.md) — what labels are and how streams are formed
- [structured-metadata](structured-metadata.md) — full guide to the structured-metadata feature
- [multi-tenancy](multi-tenancy.md) — per-tenant overrides live in the same limits config
