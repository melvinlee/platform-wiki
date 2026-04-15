---
title: Structured Metadata in Loki
summary: What structured metadata is, when to use it instead of labels, how to enable and emit it, and how to query it (including the UI quirks).
updated: 2026-04-15
sources: Grafana Labs, Loki docs (undated); jose (Canonical / Charmhub), 2024-09-06; Rudi Martinsen, 2024-12-18; Nawaz Dhandala (OneUptime), 2026-01-21
raw: [grafana-understand-labels](../../raw/grafana-loki/grafana-understand-labels.md); [solving-high-cardinality-labels-in-loki](../../raw/grafana-loki/2024-09-06-solving-high-cardinality-labels-in-loki.md); [working-with-structured-metadata-in-grafana-loki](../../raw/grafana-loki/2024-12-18-working-with-structured-metadata-in-grafana-loki.md); [how-to-optimize-loki-label-cardinality](../../raw/grafana-loki/2026-01-21-how-to-optimize-loki-label-cardinality.md)
---

# Structured Metadata in Loki

**Structured metadata** attaches key-value pairs to individual log entries **without indexing them as labels and without creating new streams**. It was added in chunk format v4, became GA in Loki 2.9.4, and is the intended home for high-cardinality fields you still want to filter on.

In short: if a field's values are too varied to safely use as a label, but you need to query by it anyway, this is the feature.

## When to use it

Grafana's documentation lists four scenarios:

1. **Native ingestion of OpenTelemetry data** — OTel resource attributes that don't fit the default index-label set go here automatically.
2. **High-cardinality metadata that's not already in the log line** — trace IDs, request IDs, pod names, filenames in per-VM scrapers. All the fields that would wreck cardinality as labels.
3. **Using Explore Logs in Grafana** to visualize and browse.
4. **Large-scale deployments using Bloom filters** for query acceleration — structured metadata is what blooms index.

Contrast with the two other options:

- **Label** — indexed, defines a stream. Use only for low-cardinality, bounded values describing log *origin*.
- **Log content** — unindexed; filter with `| json | field="…"` or similar. Free but slower, and the field has to actually be in the log line.
- **Structured metadata** — per-entry indexed, but does **not** create new streams. Best of both when the field would explode cardinality as a label.

See [cardinality](cardinality.md) for the bigger picture and [labels](labels.md) for why labels should be bounded.

## Enable it

Two config requirements:

```yaml
limits_config:
  allow_structured_metadata: true

schema_config:
  configs:
    - from: '2024-09-03'
      index:
        period: 24h
        prefix: index_
      object_store: filesystem
      schema: v13          # minimum required for structured metadata
      store: tsdb
```

Without `allow_structured_metadata: true`, Loki rejects pushes that include it. Schema must be at least v13.

## Emit it from the collector

The pipeline stage is `structured_metadata:` — it works exactly like `labels:` but writes to the per-entry metadata instead of the label set.

### Promtail / Grafana Agent — labeldrop pattern

When you already extract a field for other reasons and don't want it as a label, promote it to structured metadata and drop it from labels:

```yaml
scrape_configs:
  - job_name: varlog_scraper
    pipeline_stages:
      - drop:
          expression: .*file is a directory.*
      - structured_metadata:
          filename: filename
      - labeldrop:
          - filename
```

### Promtail — side-by-side with labels

If a regex or JSON stage has already extracted named groups, you can split them: some go to `labels:`, others go to `structured_metadata:`. Example from an OPNsense firewall log pipeline, where most extracted fields become labels but the high-cardinality `fw_sequence` is kept as metadata:

```yaml
pipeline_stages:
  - match:
      selector: '{syslog_app_name="filterlog"} !~ ".*icmp.*"'
      stages:
        - regex:
            expression: '^(?s)(?P<fw_rule>\d+),,,(?P<fw_rid>.+?),...,(?P<fw_sequence>\d+)?'
        - labels:
            fw_src:
            fw_dst:
            fw_action:
            fw_dstport:
            fw_proto:
            fw_interface:
        - structured_metadata:
            fw_sequence:
```

No separate extraction step is needed — the named group from the regex is already in scope when `structured_metadata:` runs.

## Query it

Structured metadata is **not** a stream selector. This is the most common gotcha.

**Works** — use `|` (label filter expression) after the stream selector:

```logql
{job="varlog_scraper"} | filename="/var/log/pepe.log"
{syslog_app_name="filterlog"} | fw_sequence="12345"
```

**Does not work** — putting it inside `{}`:

```logql
{job="varlog_scraper", filename="/var/log/pepe.log"}   # empty result
```

The stream selector only matches index labels, not structured metadata. Grafana's UI does not currently distinguish which fields are indexed labels vs structured metadata in autocompletion, so values appear in the label browser but can't be clicked in to filter the stream selector — they only work via the "Label filter expression" stage in the query builder.

You can verify a field is structured metadata (not a label) by putting it in `{}` and confirming the result is empty.

## Use in Grafana dashboards

Structured metadata works with Grafana dashboard variables the same way labels do, provided you reference the variable inside a `| field="$var"` filter expression rather than the stream selector.

Typical pattern:

1. Define a dashboard variable (e.g. `$sequence`).
2. In each panel query, add the filter: `{syslog_app_name="filterlog"} | fw_sequence="$sequence"`.
3. All panels on the dashboard then filter by the selected value.

## Tradeoffs

- **Queryable but not indexed for stream selection.** Filtering requires iterating log entries in the matching streams, so it's closer to content filtering than label filtering in cost. The win is that it doesn't multiply streams.
- **No cardinality limit.** Unlike labels (default max 15), structured metadata has no stream-explosion risk, but you're still paying storage for whatever you include.
- **Per-entry, not per-stream.** The same stream can carry entries with different structured-metadata values — this is exactly the opposite of labels.

## See also

- [cardinality](cardinality.md) — why high cardinality breaks Loki; structured metadata is the main remediation
- [labels](labels.md) — when to use labels vs structured metadata vs log content
