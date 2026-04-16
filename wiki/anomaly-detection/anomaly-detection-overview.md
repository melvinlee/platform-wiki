---
title: Anomaly Detection Overview
summary: Types of anomalies, why detection matters for observability, best practices for enabling AI/ML-based detection, commercial platform capabilities, and emerging trends.
updated: 2026-04-16
sources: Bijit Ghosh, 2023-05-03; Reza Rajabi, 2023-03-27
raw: [aiml-anomaly-detection-reducing-mttr-observability](../../raw/anomaly-detection/2023-05-03-aiml-anomaly-detection-reducing-mttr-observability.md); [outlier-anomaly-detection-facebook-prophet-python](../../raw/anomaly-detection/2023-03-27-outlier-anomaly-detection-facebook-prophet-python.md)
---

# Anomaly Detection Overview

Anomaly detection identifies data points, events, or observations that deviate from a dataset's normal behaviour. In observability, it replaces manual threshold-based alerting with ML models that learn baseline patterns and flag deviations automatically — enabling faster incident detection and reduced Mean Time To Resolution (MTTR).

## Types of anomalies

| Type | Description | Example |
| --- | --- | --- |
| **Point anomaly** | A single data point with an extreme value relative to the global distribution. | A sudden CPU spike to 100% on a host that normally runs at 20%. |
| **Contextual (conditional) anomaly** | A value that is normal globally but anomalous given its context (e.g. time of day, season). | High traffic at 3 AM that would be normal at noon. |
| **Collective anomaly** | A cluster or sequence of data points that are individually normal but together indicate a problem. | A slow, steady memory climb over 48 hours before an OOM. |

## Why it matters

- **Automated KPI analysis** — ML scans all metrics 24/7, surfacing deviations that manual review would miss.
- **Security breach detection** — anomalous patterns in logs/metrics can signal intrusion before damage spreads.
- **Hidden performance opportunities** — deviations from expected behaviour can reveal bottlenecks or underutilized capacity.
- **Faster results** — ML detects anomalies immediately vs hours of manual triage.

## Best practices for enabling AI/ML-based detection

1. **Collect diverse data.** Pull from logs, metrics, and traces to give the model a comprehensive view.
2. **Establish baselines.** Analyze historical data to define normal behaviour before enabling detection.
3. **Choose the right algorithm.** Match the algorithm to the data shape — statistical methods (Prophet, ARIMA) for seasonal time series, isolation forests for multidimensional metrics, Elasticsearch ML for log-centric pipelines.
4. **Refine models continuously.** Periodically review false-positive rates and retrain as system behaviour evolves.

## Implementation approaches

### Elasticsearch ML

Elasticsearch has built-in ML anomaly detection. The workflow:

1. Index time-series data (e.g. temperature readings, request counts).
2. Define a job with `bucket_span`, detectors (count, mean, etc.), and influencers.
3. Start a datafeed pointing at the index.
4. Visualize results in Kibana's Machine Learning tab.

Useful for teams already running the Elastic stack — no separate ML infrastructure needed.

### Facebook/Meta Prophet

Prophet handles time-series forecasting with built-in trend and seasonality detection. The anomaly detection pattern:

1. Fit Prophet on historical data.
2. Forecast expected values with confidence intervals.
3. Flag points where prediction error exceeds a threshold (e.g. 1.5× the uncertainty band, or 2σ of prediction errors).

See [prophet-anomaly-detection](prophet-anomaly-detection.md) for the full walkthrough and code.

### Commercial platforms

| Platform | Key capabilities |
| --- | --- |
| **New Relic** | Applied Intelligence (ML-based correlation), New Relic AI (multi-source root-cause analysis), NRQL custom alerting. |
| **Google Cloud Operations** | Built-in anomaly detection on metrics, customizable alerting and notification channels. |
| **Datadog** | ML anomaly detection on metrics/events, custom alerting, predictive analytics for proactive issue prevention. |

All three follow the same pattern: ingest telemetry, learn baselines, alert on deviations. The differentiator is integration depth with your existing stack.

## Emerging trends

- **Auto-remediation** — coupling anomaly detection with automated runbooks that fix known issues without human intervention.
- **Explainable AI** — models that explain *why* they flagged an anomaly, building trust and making it easier to act on alerts.
- **Real-time streaming detection** — analysing high-volume event streams in real time rather than on stored batches.

## See also

- [prophet-anomaly-detection](prophet-anomaly-detection.md) — hands-on guide to using Prophet for time-series anomaly detection
