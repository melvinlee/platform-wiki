# platform-wiki

[![Last commit](https://img.shields.io/github/last-commit/melvinlee/platform-wiki)](https://github.com/melvinlee/platform-wiki/commits/main)
[![Repo size](https://img.shields.io/github/repo-size/melvinlee/platform-wiki)](https://github.com/melvinlee/platform-wiki)
[![Articles](https://img.shields.io/badge/articles-4-blue)](wiki/index.md)
[![Sources](https://img.shields.io/badge/raw%20sources-7-orange)](raw/)

A personal knowledge base compiled from raw sources by the [karpathy-llm-wiki](.claude/skills/karpathy-llm-wiki/SKILL.md) skill.

- **[raw/](raw/)** — immutable source material, organized by topic
- **[wiki/](wiki/)** — compiled knowledge articles
- **[.claude/skills/karpathy-llm-wiki/](.claude/skills/karpathy-llm-wiki/)** — the skill that maintains the wiki

## Index

### grafana-loki

How Grafana Loki ingests, indexes, and isolates log data.

| Article | Summary | Updated |
|---------|---------|---------|
| [Labels in Loki](wiki/grafana-loki/labels.md) | How Loki uses labels to define log streams, what makes a good label, and how streams are created. | 2026-04-15 |
| [Cardinality in Loki](wiki/grafana-loki/cardinality.md) | Why high label cardinality breaks Loki, how to detect it, and how to fix it with structured metadata and label discipline. | 2026-04-15 |
| [Structured Metadata in Loki](wiki/grafana-loki/structured-metadata.md) | What structured metadata is, when to use it instead of labels, how to enable and emit it, and how to query it (including the UI quirks). | 2026-04-15 |
| [Multi-tenancy in Loki](wiki/grafana-loki/multi-tenancy.md) | How Loki isolates tenant data with X-Scope-OrgID, and how to wire up auth, reverse proxies, Promtail pipelines, and Grafana datasources. | 2026-04-15 |

See [wiki/index.md](wiki/index.md) for the canonical index and [wiki/log.md](wiki/log.md) for the operation history.
