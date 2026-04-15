---
title: Labels in Loki
summary: How Loki uses labels to define log streams, what makes a good label, and how streams are created.
updated: 2026-04-15
sources: Grafana Labs, Loki docs (undated); Nawaz Dhandala (OneUptime), 2026-01-21
raw: [grafana-understand-labels](../../raw/grafana-loki/grafana-understand-labels.md); [how-to-optimize-loki-label-cardinality](../../raw/grafana-loki/2026-01-21-how-to-optimize-loki-label-cardinality.md)
---

# Labels in Loki

Labels are the indexing primitive in Loki. The content of log lines is **not** indexed — instead, logs are grouped into **streams** and streams are indexed by their label set. A stream is uniquely identified by the combination of every label key and value; changing one value creates a new stream.

Each log stream must have at least one label to be stored and queried.

## How a stream is formed

A label is a key-value pair, e.g.:

- `deployment_environment=development`
- `cloud_region=us-west-1`
- `namespace=grafana-server`

A set of log messages that share all labels is a single log stream. Queries first locate the matching streams via the index, then iterate log lines within those streams for content filtering.

Example — a pipeline that extracts `action` and `status_code` from Apache logs will produce one stream per unique `(action, status_code)` combination. Four lines with `GET/200`, `POST/200`, `GET/400`, `POST/400` yield four separate streams and four chunks. Adding an `ip` label would multiply this by the number of unique IPs.

## Default labels

Loki tries to populate a default `service_name` label on ingest. It looks for the first match in this ordered list:

`service_name`, `service`, `app`, `application`, `name`, `app_kubernetes_io_name`, `container`, `container_name`, `component`, `workload`, `job`.

If nothing matches, it uses `unknown_service`. This list is configurable via `discover_service_name` in `limits_config`.

### OpenTelemetry defaults

When Alloy or the OpenTelemetry Collector is the client, Loki promotes certain OTel resource attributes to labels (with `.` replaced by `_`); the rest go to structured metadata. Defaults include: `cloud.region`, `cloud.availability_zone`, `container.name`, `deployment.environment.name`, `k8s.cluster.name`, `k8s.container.name`, `k8s.namespace.name`, `k8s.pod.name`, `service.name`, `service.namespace`, `service.instance.id`, and related workload attributes.

Loki has a default limit of 15 index labels; the default OTel list exceeds this but many entries are mutually exclusive. `k8s.pod.name` and `service.instance.id` are **no longer recommended** as default labels due to cardinality — new users should convert them to structured metadata.

## What makes a good label

Labels should describe the *source* of logs, not their *content*. Good candidates:

- Environment: `env=prod`, `env=staging` — fixed, small set
- Service/app: `service`, `job`, `app` — bounded set of services
- Log level: `level=info|warn|error` — fixed, small set
- Namespace / team — bounded
- Region / cluster — fixed set
- Hostname — only if hosts are stable; for ephemeral VMs/pods, use structured metadata instead
- Filename — only if bounded; not for cases like per-VM log files

Principles from the Grafana docs:

- **DO** keep the label set small — aim for ≤ 10–15.
- **DO** prefer long-lived, bounded values. A label whose values churn rapidly produces short-lived streams.
- **DO** base labels on what users actually query.
- **DON'T** label content-level attributes (message text, exception names, request IDs, user IDs).
- **DON'T** create labels for rare ad-hoc searches.

### Label format

Loki follows Prometheus naming rules: `[a-zA-Z_:][a-zA-Z0-9_:]*`. Unsupported characters in source attribute names are converted to underscores (e.g. `app.kubernetes.io/name` → `app_kubernetes_io_name`). Names must not begin and end with double underscores — that pattern is reserved for internal labels like `__stream_shard__`.

## Labeling is iterative

Start with a small label set (the defaults from Alloy / Kubernetes Monitoring Helm chart are usually a good base). Expect to refine over time as query patterns emerge. Changing labels later is normal — settling on the right set often takes several rounds of testing.

## Labels and out-of-order ingestion

Out-of-order writes are enabled globally by default but still have a per-stream constraint: within a given stream, entries must arrive within a two-hour window. Entries too old for a stream are rejected with "too far behind."

If your sources have very different ingestion delays, separate them with a label so each gets its own stream. For example, instead of `{environment="production"}` for everything, split into `{environment="production", app="slow_app"}` and `{environment="production", app="fast_app"}`.

## When not to use labels

If you need to filter on a high-cardinality or unbounded attribute (user ID, request ID, trace ID, filename in a per-VM scheme), don't promote it to a label. Options:

1. Leave it in the log line and filter with `| json | user_id="…"` at query time.
2. Promote it to **structured metadata**, which is indexed per-entry without creating new streams (requires Loki 2.9.4+ and schema v13, with `allow_structured_metadata: true`).

See [cardinality](cardinality.md) for the full rationale and remediation patterns.

## See also

- [cardinality](cardinality.md) — why high cardinality breaks Loki and how to fix it
- [multi-tenancy](multi-tenancy.md) — tenant isolation via `X-Scope-OrgID`
