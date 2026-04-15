---
title: "How to Optimize Loki Label Cardinality"
source: "https://oneuptime.com/blog/post/2026-01-21-loki-label-cardinality/view"
author:
  - "[[Nawaz Dhandala]]"
published: 2026-01-21
created: 2026-04-15
description: "A comprehensive guide to optimizing label cardinality in Grafana Loki, covering label selection strategies, high-cardinality detection, remediation techniques."
tags:
  - "clippings"
---
---

Label cardinality - the number of unique label value combinations - directly impacts Loki's performance and cost. High cardinality creates excessive streams, increases index size, degrades query performance, and can lead to ingestion failures. This guide explains how to identify and fix cardinality issues for optimal Loki operation.

## Understanding Label Cardinality

### What is Cardinality?

Cardinality refers to the number of unique values for a label or unique combinations of all labels (streams):

```
TextLabels: {job="api", instance="pod-1", env="prod"}

Cardinality examples:
- job: 5 unique values (low cardinality - good)
- instance: 100 unique values (medium cardinality - acceptable)
- request_id: millions of unique values (high cardinality - BAD)
```

### Why High Cardinality is Problematic

<svg id="mermaid-1776256523145" width="100%" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" style="max-width: 2407.6015625px;" viewBox="-7.5 -8 2407.6015625 175" role="graphics-document document" aria-roledescription="flowchart-v2"><g><marker id="mermaid-1776256523145_flowchart-pointEnd" viewBox="0 0 10 10" refX="6" refY="5" markerUnits="userSpaceOnUse" markerWidth="12" markerHeight="12" orient="auto"><path d="M 0 0 L 10 5 L 0 10 z" style="stroke-width: 1; stroke-dasharray: 1, 0;"></path></marker><marker id="mermaid-1776256523145_flowchart-pointStart" viewBox="0 0 10 10" refX="4.5" refY="5" markerUnits="userSpaceOnUse" markerWidth="12" markerHeight="12" orient="auto"><path d="M 0 5 L 10 10 L 10 0 z" style="stroke-width: 1; stroke-dasharray: 1, 0;"></path></marker><marker id="mermaid-1776256523145_flowchart-circleEnd" viewBox="0 0 10 10" refX="11" refY="5" markerUnits="userSpaceOnUse" markerWidth="11" markerHeight="11" orient="auto"><circle cx="5" cy="5" r="5" style="stroke-width: 1; stroke-dasharray: 1, 0;"></circle></marker><marker id="mermaid-1776256523145_flowchart-circleStart" viewBox="0 0 10 10" refX="-1" refY="5" markerUnits="userSpaceOnUse" markerWidth="11" markerHeight="11" orient="auto"><circle cx="5" cy="5" r="5" style="stroke-width: 1; stroke-dasharray: 1, 0;"></circle></marker><marker id="mermaid-1776256523145_flowchart-crossEnd" viewBox="0 0 11 11" refX="12" refY="5.2" markerUnits="userSpaceOnUse" markerWidth="11" markerHeight="11" orient="auto"><path d="M 1,1 l 9,9 M 10,1 l -9,9" style="stroke-width: 2; stroke-dasharray: 1, 0;"></path></marker><marker id="mermaid-1776256523145_flowchart-crossStart" viewBox="0 0 11 11" refX="-1" refY="5.2" markerUnits="userSpaceOnUse" markerWidth="11" markerHeight="11" orient="auto"><path d="M 1,1 l 9,9 M 10,1 l -9,9" style="stroke-width: 2; stroke-dasharray: 1, 0;"></path></marker><g><g></g><g></g><g></g><g><g transform="translate(-7.5, -8)"><g><g id="Impact"><rect style="" rx="0" ry="0" x="8" y="8" width="2391.6015625" height="159"></rect><g transform="translate(1105.546875, 8)"><foreignObject width="196.5078125" height="24"><div xmlns="http://www.w3.org/1999/xhtml" style="display: inline-block; white-space: nowrap;">Impact of High Cardinality</div></foreignObject></g></g></g><g><path d="M894.656,87.5L898.823,87.5C902.99,87.5,911.323,87.5,918.773,87.5C926.223,87.5,932.79,87.5,936.073,87.5L939.356,87.5" id="L-Effects-Formula-0" style="fill:none;" marker-end="url(#mermaid-1776256523145_flowchart-pointEnd)" stroke="currentColor"></path></g><g><g><g transform="translate(0, 0)"></g></g></g><g><g transform="translate(25.5, 35)"><g><g id="Effects"><rect style="" rx="0" ry="0" x="8" y="8" width="861.65625" height="89"></rect><g transform="translate(305.3671875, 8)"><foreignObject width="266.921875" height="24"><div xmlns="http://www.w3.org/1999/xhtml" style="display: inline-block; white-space: nowrap;">More Streams = More Index Entries</div></foreignObject></g></g></g><g></g><g></g><g><g id="flowchart-E1-0" data-node="true" data-id="E1" transform="translate(115.015625, 52.5)"><rect style="" rx="0" ry="0" x="-72.015625" y="-19.5" width="144.03125" height="39" fill="none" stroke="currentColor"></rect><g style="" transform="translate(-64.515625, -12)"><rect></rect><foreignObject width="129.03125" height="24"><div xmlns="http://www.w3.org/1999/xhtml" style="display: inline-block; white-space: nowrap;">Larger index size</div></foreignObject></g></g><g id="flowchart-E2-1" data-node="true" data-id="E2" transform="translate(300.7890625, 52.5)"><rect style="" rx="0" ry="0" x="-63.7578125" y="-19.5" width="127.515625" height="39" fill="none" stroke="currentColor"></rect><g style="" transform="translate(-56.2578125, -12)"><rect></rect><foreignObject width="112.515625" height="24"><div xmlns="http://www.w3.org/1999/xhtml" style="display: inline-block; white-space: nowrap;">Slower queries</div></foreignObject></g></g><g id="flowchart-E3-2" data-node="true" data-id="E3" transform="translate(505.8671875, 52.5)"><rect style="" rx="0" ry="0" x="-91.3203125" y="-19.5" width="182.640625" height="39" fill="none" stroke="currentColor"></rect><g style="" transform="translate(-83.8203125, -12)"><rect></rect><foreignObject width="167.640625" height="24"><div xmlns="http://www.w3.org/1999/xhtml" style="display: inline-block; white-space: nowrap;">Higher memory usage</div></foreignObject></g></g><g id="flowchart-E4-3" data-node="true" data-id="E4" transform="translate(740.921875, 52.5)"><rect style="" rx="0" ry="0" x="-93.734375" y="-19.5" width="187.46875" height="39" fill="none" stroke="currentColor"></rect><g style="" transform="translate(-86.234375, -12)"><rect></rect><foreignObject width="172.46875" height="24"><div xmlns="http://www.w3.org/1999/xhtml" style="display: inline-block; white-space: nowrap;">More S3/GCS API calls</div></foreignObject></g></g></g></g><g transform="translate(937.15625, 35)"><g><g id="Formula"><rect style="" rx="0" ry="0" x="8" y="8" width="1429.9453125" height="89"></rect><g transform="translate(626.8046875, 8)"><foreignObject width="192.3359375" height="24"><div xmlns="http://www.w3.org/1999/xhtml" style="display: inline-block; white-space: nowrap;">Stream Count Calculation</div></foreignObject></g></g></g><g></g><g></g><g><g id="flowchart-F1-4" data-node="true" data-id="F1" transform="translate(228.90234375, 52.5)"><rect style="" rx="0" ry="0" x="-185.90234375" y="-19.5" width="371.8046875" height="39" fill="none" stroke="currentColor"></rect><g style="" transform="translate(-178.40234375, -12)"><rect></rect><foreignObject width="356.8046875" height="24"><div xmlns="http://www.w3.org/1999/xhtml" style="display: inline-block; white-space: nowrap;">Stream count = Product of all label cardinalities</div></foreignObject></g></g><g id="flowchart-F2-5" data-node="true" data-id="F2" transform="translate(703.15234375, 52.5)"><rect style="" rx="0" ry="0" x="-238.34765625" y="-19.5" width="476.6953125" height="39" fill="none" stroke="currentColor"></rect><g style="" transform="translate(-230.84765625, -12)"><rect></rect><foreignObject width="461.6953125" height="24"><div xmlns="http://www.w3.org/1999/xhtml" style="display: inline-block; white-space: nowrap;">✅ Example: job(5) x instance(100) x level(4) = 2,000 streams</div></foreignObject></g></g><g id="flowchart-F3-6" data-node="true" data-id="F3" transform="translate(1197.22265625, 52.5)"><rect style="" rx="0" ry="0" x="-205.72265625" y="-19.5" width="411.4453125" height="39" fill="none" stroke="currentColor"></rect><g style="" transform="translate(-198.22265625, -12)"><rect></rect><foreignObject width="396.4453125" height="24"><div xmlns="http://www.w3.org/1999/xhtml" style="display: inline-block; white-space: nowrap;">❌ Bad: job(5) x user_id(100,000) = 500,000 streams</div></foreignObject></g></g></g></g></g></g></g></g></g></svg>

## Identifying High Cardinality

### Check Stream Count

```bash
Bash# Current stream count per tenant

curl -s 'http://loki:3100/loki/api/v1/label/__name__/values' | jq

# Active streams metric
curl -s http://loki:3100/metrics | grep "loki_ingester_streams_created_total"
```

### Prometheus Queries for Cardinality

```
Promql# Total active streams
sum(loki_ingester_memory_streams)

# Streams per tenant
sum by (tenant) (loki_ingester_memory_streams)

# Stream creation rate (high rate indicates cardinality explosion)
rate(loki_ingester_streams_created_total[5m])

# Streams per job
sum by (job) (loki_ingester_memory_streams)
```

### LogQL Cardinality Analysis

```
Logql# Count unique values for a label
count(count by (user_id) (rate({job="app"}[1h])))

# Find high-cardinality label values
topk(20, sum by (pod) (count_over_time({job="app"}[1h])))

# Identify labels causing stream explosion
sum by (request_id) (count_over_time({job="app"}[5m]))
```

### Check Loki Logs for Warnings

```bash
Bash# Find cardinality warnings
docker logs loki 2>&1 | grep -i "cardinality\|streams\|rate limit"

# Check for stream limit errors
docker logs loki 2>&1 | grep "max streams"
```

## Common High-Cardinality Labels

### Labels to AVOID as Loki Labels

| Label Type | Example | Why Bad |
| --- | --- | --- |
| Request IDs | request\_id, trace\_id | Unique per request |
| User IDs | user\_id, customer\_id | Unbounded growth |
| Timestamps | log\_time, event\_time | Already indexed by Loki |
| Session IDs | session\_id | Short-lived, unique |
| IP Addresses | client\_ip, source\_ip | Many unique values |
| Full Paths | file\_path, url | Many variations |

### Labels that ARE Appropriate

| Label Type | Example | Why Good |
| --- | --- | --- |
| Environment | env=prod, env=staging | Few fixed values |
| Service Name | service, app | Known set of services |
| Log Level | level=info, level=error | Fixed set (4-5 values) |
| Namespace | namespace, team | Bounded set |
| Node/Instance | node, instance | Limited by infrastructure |
| Region | region, datacenter | Fixed set |

## Remediation Strategies

### Move High-Cardinality Data to Log Content

**Bad - user\_id as label:**

```yaml
YAML# promtail-config.yaml - DON'T DO THIS
pipeline_stages:
  - json:
      expressions:
        user_id: user_id
  - labels:
      user_id:  # Creates millions of streams!
```

**Good - user\_id in log content:**

```yaml
YAML# promtail-config.yaml - DO THIS
pipeline_stages:
  - json:
      expressions:
        user_id: user_id
  # Don't add user_id as label
  # Query with: {job="app"} | json | user_id="12345"
```

### Use Structured Metadata (Loki 2.7+)

```yaml
YAML# promtail-config.yaml
pipeline_stages:
  - json:
      expressions:
        trace_id: trace_id
        user_id: user_id
  # Store as structured metadata instead of labels
  - structured_metadata:
      trace_id:
      user_id:
```
```yaml
YAML# loki-config.yaml - Enable structured metadata
limits_config:
  allow_structured_metadata: true
```

Query with structured metadata:

```
Logql# Filter by structured metadata
{job="app"} | trace_id="abc123"

# Structured metadata is indexed but doesn't create streams
```

### Aggregate or Bucket Values

**Bad - exact request duration as label:**

```yaml
YAMLpipeline_stages:
  - json:
      expressions:
        duration_ms: duration
  - labels:
      duration_ms:  # Millions of unique values!
```

**Good - bucketed duration:**

```yaml
YAMLpipeline_stages:
  - json:
      expressions:
        duration_ms: duration
  - template:
      source: duration_bucket
      template: '{{ if lt .duration_ms 100 }}fast{{ else if lt .duration_ms 1000 }}medium{{ else }}slow{{ end }}'
  - labels:
      duration_bucket:  # Only 3 values
```

### Drop Labels Before Ingestion

```yaml
YAML# promtail-config.yaml
pipeline_stages:
  - json:
      expressions:
        level: level
        service: service
        request_id: request_id  # Extract but don't use as label
  - labels:
      level:
      service:
      # Don't add request_id as label

  # Or explicitly drop if added by relabeling
  - labeldrop:
      - request_id
      - trace_id
      - user_id
```

### Relabel with Limits

```yaml
YAML# promtail-config.yaml - Kubernetes example
relabel_configs:
  # Keep only necessary labels
  - action: labelkeep
    regex: (job|namespace|pod|container|node)

  # Drop high-cardinality labels
  - action: labeldrop
    regex: (pod_uid|controller_revision_hash|pod_template_hash)
```

## Configure Cardinality Limits

### Loki Configuration

```yaml
YAML# loki-config.yaml
limits_config:
  # Maximum streams per user/tenant
  max_streams_per_user: 10000
  max_global_streams_per_user: 50000

  # Maximum label names per series
  max_label_names_per_series: 15

  # Maximum label name length
  max_label_name_length: 1024

  # Maximum label value length
  max_label_value_length: 2048

  # Maximum labels size
  max_labels_size_bytes: 4096
```

### Per-Tenant Overrides

```yaml
YAMLoverrides:
  # Strict limits for problematic tenant
  high-volume-tenant:
    max_streams_per_user: 5000
    max_label_names_per_series: 10

  # Higher limits for platform team
  platform:
    max_streams_per_user: 50000
    max_label_names_per_series: 20
```

## Monitoring Cardinality

### Prometheus Alerts

```yaml
YAMLgroups:
  - name: loki-cardinality
    rules:
      - alert: LokiHighStreamCount
        expr: |
          sum by (tenant) (loki_ingester_memory_streams) > 50000
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High stream count for tenant {{ $labels.tenant }}"
          description: "Stream count: {{ $value }}"

      - alert: LokiStreamCreationSpike
        expr: |
          rate(loki_ingester_streams_created_total[5m]) > 100
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Rapid stream creation detected"
          description: "Creating {{ $value }} streams/sec"

      - alert: LokiMaxStreamsReached
        expr: |
          loki_discarded_samples_total{reason="per_user_series_limit"} > 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Stream limit reached - logs being dropped"
```

### Cardinality Dashboard

```json
JSON{
  "dashboard": {
    "title": "Loki Label Cardinality",
    "panels": [
      {
        "title": "Total Active Streams",
        "type": "stat",
        "targets": [
          {
            "expr": "sum(loki_ingester_memory_streams)"
          }
        ]
      },
      {
        "title": "Streams by Tenant",
        "type": "piechart",
        "targets": [
          {
            "expr": "sum by (tenant) (loki_ingester_memory_streams)",
            "legendFormat": "{{tenant}}"
          }
        ]
      },
      {
        "title": "Stream Creation Rate",
        "type": "timeseries",
        "targets": [
          {
            "expr": "sum(rate(loki_ingester_streams_created_total[5m]))",
            "legendFormat": "Streams/s"
          }
        ]
      },
      {
        "title": "Streams Rejected (Limit)",
        "type": "timeseries",
        "targets": [
          {
            "expr": "sum(rate(loki_discarded_samples_total{reason=\"per_user_series_limit\"}[5m]))",
            "legendFormat": "Rejected"
          }
        ]
      }
    ]
  }
}
```

## Label Design Guidelines

### Recommended Label Schema

```yaml
YAML# Good label schema
labels:
  # Environment context (low cardinality)
  env: production        # prod, staging, dev
  region: us-east-1      # Few regions
  cluster: main          # Few clusters

  # Service identity (bounded)
  job: api-server        # Service name
  namespace: payments    # Team/domain
  app: checkout          # Application name

  # Instance (bounded by infrastructure)
  instance: pod-abc123   # Acceptable if bounded

  # Log classification (fixed set)
  level: error           # debug, info, warn, error

  # DO NOT include as labels:
  # - request_id
  # - user_id
  # - trace_id
  # - session_id
  # - timestamp variations
```

### Query High-Cardinality Data

Instead of labels, query high-cardinality data from log content:

```
Logql# Find logs for specific user
{job="app"} | json | user_id="12345"

# Find logs for specific request
{job="app"} | json | request_id="abc-123-def"

# Aggregate by user (from content)
sum by (user_id) (
  count_over_time({job="app"} | json | user_id!="" [1h])
)
```

## Migration Strategy

### Identify Problematic Labels

```bash
Bash# Find highest cardinality labels
curl -s 'http://loki:3100/loki/api/v1/labels' | jq

# For each label, check cardinality
for label in $(curl -s 'http://loki:3100/loki/api/v1/labels' | jq -r '.data[]'); do
  count=$(curl -s "http://loki:3100/loki/api/v1/label/$label/values" | jq '.data | length')
  echo "$label: $count"
done
```

### Gradual Migration

1. **Identify** high-cardinality labels
2. **Update Promtail** to stop adding them as labels
3. **Update queries** to filter by content instead
4. **Wait for retention** to expire old streams
5. **Monitor** stream count reduction

## Best Practices Summary

1. **Think before labeling**: Ask "Will this have bounded values?"
2. **Use content filtering**: Query high-cardinality values from log content
3. **Structured metadata**: Use for trace IDs, request IDs (Loki 2.7+)
4. **Set limits**: Configure max\_streams\_per\_user appropriately
5. **Monitor cardinality**: Alert on stream count and creation rate
6. **Regular audits**: Review label usage periodically
7. **Document standards**: Create team guidelines for label usage

## Conclusion

Optimizing label cardinality is essential for maintaining Loki performance and controlling costs. By carefully selecting which values become labels and moving high-cardinality data to log content or structured metadata, you can achieve efficient indexing while retaining full queryability.

Key takeaways:

- Only use labels for low-cardinality, bounded values
- Move high-cardinality data to log content
- Use structured metadata for trace/request IDs
- Configure stream limits to catch cardinality explosions
- Monitor stream counts and creation rates
- Query high-cardinality values with content filters

@nawazdhandala • Jan 21, 2026 •

Nawaz is building OneUptime with a passion for engineering reliable systems and improving observability.

[GitHub](https://github.com/nawazdhandala)

### Improve this Blog Post

All our blog posts are open source. Found a typo, want to add more detail, or have a better explanation? Anyone can contribute and make this post better for everyone.

[Edit this Post on GitHub](https://github.com/oneuptime/blog/tree/master/posts/2026-01-21-loki-label-cardinality) [Contributing Guidelines](https://github.com/oneuptime/blog)