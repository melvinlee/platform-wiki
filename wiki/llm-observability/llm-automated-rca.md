---
title: Automating Root Cause Analysis with LLMs and MCP
summary: Architecture for automated RCA using Golden Signals alerts, LLM agents, and Model Context Protocol (MCP) tools to fetch monitoring/logging data and generate root-cause analysis.
updated: 2026-04-16
sources: Jheel Patel, 2025-03-25
raw: [automating-rca-with-llms-and-mcp](../../raw/llm-observability/2025-03-25-automating-rca-with-llms-and-mcp.md)
---

# Automating Root Cause Analysis with LLMs and MCP

Root cause analysis (RCA) is traditionally manual: an engineer gets paged, opens dashboards, queries logs, correlates metrics, and writes up findings. This article describes an architecture that automates the process by combining **Golden Signals alerting**, **LLM agents**, and **MCP (Model Context Protocol)** tools that fetch monitoring and logging data on demand.

The key insight: instead of pre-building rigid data-aggregation pipelines for every failure mode, give the LLM the *tools* to pull data dynamically and let it decide what to fetch based on the alert context.

## The Four Golden Signals

SRE's recommended monitoring signals for user-facing services:

| Signal | What it measures |
| --- | --- |
| **Latency** | Time to serve a request (distinguish success vs error latency) |
| **Traffic** | Demand on the system (requests/sec, sessions, etc.) |
| **Errors** | Rate of failed requests (HTTP 5xx, timeouts, etc.) |
| **Saturation** | How "full" the service is (CPU, memory, disk, queue depth) |

When these signal an anomaly, the system triggers an alert — and that alert is the entry point for automated RCA.

## Architecture

```
Golden Signals alert
    → webhook
        → MCP Client (LLM agent)
            → MCP Server (tools)
                → Prometheus / Cloud Monitoring (PromQL queries)
                → Cloud Logging / log store (log queries)
            ← data
        ← RCA report (root cause + suggestions)
```

The flow:

1. Alert policies monitor Golden Signals (e.g. high latency on a load balancer, 5xx error rate spike).
2. When an alert fires, its payload is sent to a **webhook** that invokes the MCP Client.
3. The MCP Client — an **LLM agent** (e.g. Claude) — receives the alert payload and decides which tools to call.
4. The **MCP Server** exposes tools the LLM can invoke:
   - **Prometheus Query Tool**: accepts PromQL queries + their purpose, fetches monitoring data.
   - **Logging Query Tool**: accepts a log query (e.g. Cloud Logging filter), fetches relevant error logs.
5. The LLM analyzes the returned data and generates a root-cause analysis with resolution suggestions.

The LLM prompt is crafted to use tools judiciously — only fetch data when needed, to minimize token usage and API calls.

## What MCP provides

[Model Context Protocol](https://modelcontextprotocol.io/introduction) (open-sourced by Anthropic) gives LLMs structured access to external tools and data sources. Think of it as "hands and eyes" for the model — the ability to interact with its environment rather than just processing text.

For RCA, MCP means the agent can:

- Query different monitoring backends (Prometheus, CloudWatch, Datadog) through defined tool interfaces.
- Fetch logs from different sources (Cloud Logging, Elasticsearch, Loki) as needed.
- Adapt its investigation dynamically based on what it finds, rather than following a static runbook.

## Benefits over traditional automated RCA

- **No custom parsers** — LLMs handle unstructured data natively.
- **Better edge-case handling** — both in interpreting alert payloads and processing tool results.
- **Efficient data usage** — only the necessary monitoring/logging data is fetched, only when needed.
- **No complex UIs** — eliminates manual dashboard navigation during incidents.
- **Declarative, not imperative** — you describe what good RCA looks like in a prompt, not the step-by-step logic.
- **Dynamic investigation** — the agent adapts its tool calls based on what the alert and intermediate data reveal, rather than following pre-built static flows.
- **Business logic via prompts** — operational practices and domain knowledge can be injected via prompt templates without code changes.

## Implementation considerations

### Tool design

- **Robust exception handling** with descriptive error messages — lets the LLM retry intelligently by adjusting its arguments.
- **Expressive docstrings** — the LLM uses tool descriptions to decide when and how to call them. Better descriptions = fewer wasted calls.
- **Pre-filter data** — apply server-side filtering before returning data to the LLM. Reducing irrelevant data saves tokens and improves analysis quality.

### Prompt design

There's a tradeoff between prompt complexity and tool complexity:

- A detailed prompt can compensate for simpler tools, but increases token cost on every invocation.
- More capable tools (e.g. pre-aggregating metrics) reduce what the LLM needs to reason about, but are harder to build and maintain.

The sweet spot depends on incident volume and cost constraints.

## Extensions

1. **Auto-remediation tools** — give the agent tools to execute well-understood fixes (restart a service, scale up, clear a queue) when the root cause matches a known pattern.
2. **ITSM integration** — tools to create/update/resolve incidents in ServiceNow, PagerDuty, Jira, etc.
3. **Prompt template experimentation** — different prompt structures yield different analysis quality; treat prompts as a tunable parameter.

## Reference implementation

A PoC with Google Cloud (Regional External LB, VM instance group, Cloud SQL) is available at [github.com/scienceto/mcp-rca](https://github.com/scienceto/mcp-rca). It uses Claude as the LLM, two MCP tools (Prometheus query + Cloud Logging query), and alert policies on latency and 5xx rates.

## See also

- [llm-log-analysis](llm-log-analysis.md) — LLM techniques for log parsing, structuring, summarization, and anomaly detection
- [anomaly-detection-overview](../anomaly-detection/anomaly-detection-overview.md) — ML-based anomaly detection approaches that can feed into the Golden Signals alerting layer
