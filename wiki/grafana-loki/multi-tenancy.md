---
title: Multi-tenancy in Loki
summary: How Loki isolates tenant data with X-Scope-OrgID, and how to wire up auth, reverse proxies, Promtail pipelines, and Grafana datasources for a production multi-tenant setup.
updated: 2026-04-15
sources: Grafana Labs, Loki docs (undated); Sander Rodenhuis (Otomi / Akamai App Platform), 2022-09-24
raw: [grafana-manage-tenant-isolation](../../raw/grafana-loki/grafana-manage-tenant-isolation.md); [multi-tenancy-with-loki-promtail-grafana-demystified](../../raw/grafana-loki/2022-09-24-multi-tenancy-with-loki-promtail-grafana-demystified.md)
---

# Multi-tenancy in Loki

Loki is a multi-tenant system: requests and data for tenant A are isolated from tenant B. The tenant for a request is identified by the `X-Scope-OrgID` HTTP header. Loki itself does not authenticate users — tenant isolation is enforced purely by this header, so an authenticating reverse proxy in front of Loki is what makes the model secure in practice.

## The core mechanism

- Loki runs in multi-tenant mode when `auth_enabled: true` (the default).
- Every API request must include `X-Scope-OrgID: <tenant>`.
- Tenant IDs are alphanumeric strings, ≤ 150 bytes. `.` and `..` are forbidden for security reasons.
- Operators should keep tenant IDs short — ~20 bytes is usually enough.
- With `auth_enabled: false`, Loki runs single-tenant and the ID is hardcoded to `fake`; the header is not required.

On ingestion, Promtail / Alloy / Grafana Agent tags each batch with a `tenant_id`, which Loki translates to the `X-Scope-OrgID` header on push. On query, the reverse proxy authenticating the user injects `X-Scope-OrgID` based on the authenticated identity.

## Multi-tenant queries

Cross-tenant queries are opt-in. Enable with:

```yaml
querier:
  multi_tenant_queries_enabled: true
```

The query request declares the tenants in the header, pipe-separated:

```
X-Scope-OrgID: A|B
```

LogQL also supports filtering by tenant ID as a stream selector label:

```logql
{app="foo", __tenant_id__=~"a.+"} | logfmt
```

returns results for all tenants whose ID starts with `a`. If the label `__tenant_id__` already exists on a stream, it is renamed to `original___tenant_id__` (prefixed with `original_`). Filtering `__tenant_id__` *in a pipeline stage* is **not** supported — this query fails:

```logql
{app="foo"} | __tenant_id__="1" | logfmt    # does not work
```

Only query endpoints accept multi-tenant calls. `GET /loki/api/v1/tail` and `POST /loki/api/v1/push` reject headers with more than one tenant (HTTP 400).

## Production architecture pattern

Loki has no concept of users or passwords. A typical production setup (e.g. the Otomi / Akamai App Platform reference) looks like this:

1. **OAuth2 / identity broker** in front of Grafana — users must belong to a tenant group to access the tenant's Grafana instance. A JWT is passed to Grafana.
2. **Per-tenant Grafana** in each tenant namespace. Admins provision the Loki datasource centrally; tenants cannot edit data sources.
3. **Authenticating reverse proxy** runs as a sidecar next to Loki in the same pod (e.g. `k8spin/loki-multi-tenant-proxy`). It validates basic auth credentials and injects the matching `X-Scope-OrgID` header, then forwards to Loki on localhost.
4. **Loki** runs with `auth_enabled: true` and relies on the proxy for identity.
5. **Promtail / Alloy** tags every log with a `tenant` value based on the log's namespace (or similar metadata).

### Promtail: tagging logs with their tenant

Tenants are admin-scoped in this example — Promtail itself pushes to Loki without user/password because it lives in an isolated monitoring namespace.

```yaml
# promtail values.yaml
config:
  clients:
    - url: http://loki.monitoring:3100/loki/api/v1/push
      tenant_id: admins
```

Use a pipeline `match` stage keyed on namespace to rewrite the tenant per log line:

```yaml
snippets:
  pipelineStages:
    - cri: {}
    - json:
        expressions:
          namespace:
    - labels:
        namespace:
    - match:
        selector: '{namespace="tenant-1"}'
        stages:
          - tenant:
              value: tenant-1
    - match:
        selector: '{namespace="tenant-2"}'
        stages:
          - tenant:
              value: tenant-2
    - output:
        source: message
```

Promtail pipelines have four stage types: parsing (extract data), transform (modify extracted data), action (e.g. set labels, set tenant), filtering (conditional / drop). The `tenant` action stage is what sets `X-Scope-OrgID` for the affected entries.

### Loki: enable multi-tenancy and attach the sidecar proxy

```yaml
# loki values
config:
  auth_enabled: true

extraContainers:
  - name: reverse-proxy
    image: k8spin/loki-multi-tenant-proxy:v1.0.0
    args:
      - "run"
      - "--port=3101"
      - "--loki-server=http://localhost:3100"
      - "--auth-config=/etc/reverse-proxy-conf/authn.yaml"
    ports:
      - name: http
        containerPort: 3101
    volumeMounts:
      - name: reverse-proxy-auth-config
        mountPath: /etc/reverse-proxy-conf

extraVolumes:
  - name: reverse-proxy-auth-config
    secret:
      secretName: reverse-proxy-auth-config

extraPorts:
  - port: 3101
    protocol: TCP
    name: http
    targetPort: http
```

The proxy's auth file maps basic auth credentials to tenant IDs:

```yaml
# authn.yaml (mounted from a secret)
- username: admin
  password: password
  orgid: admins
- username: tenant-1
  password: password
  orgid: tenant-1
- username: tenant-2
  password: password
  orgid: tenant-2
```

Store this as a Kubernetes secret and mount it. In real deployments, encrypt with SOPS or fetch from a secret manager.

### Grafana: per-tenant datasource

Each tenant's Grafana instance is provisioned with a read-only Loki datasource pointing at the reverse proxy port (3101, **not** 3100), using the tenant's basic auth credentials:

```yaml
additionalDataSources:
  - name: Loki
    editable: false
    type: loki
    access: proxy
    url: http://loki.monitoring:3101
    basicAuth: true
    basicAuthUser: tenant-1
    secureJsonData:
      basicAuthPassword: password
```

> Version gotcha: recent Grafana versions only honor `basicAuthPassword` under `secureJsonData`. Older placements silently fail and the datasource returns empty results — a common failure mode when upgrading the Grafana Helm chart.

## Security perimeter

The tenant model is only as strong as the authentication in front of it. Loki trusts the `X-Scope-OrgID` header blindly. That means:

- Never expose Loki (port 3100) directly to tenants — only the authenticating proxy.
- Tenants must not be able to bypass the proxy or forge the header.
- The broader perimeter (OAuth2 → Istio/nginx → Grafana → reverse proxy → Loki) is what makes the model actually safe. If any link fails, the tenant boundary fails.

## Operational note

Setting this up once is straightforward. Onboarding is the hard part: each new tenant needs a namespace, a Grafana instance, encrypted credentials, an entry in the reverse proxy auth file, and a Promtail pipeline match. Production platforms (e.g. Otomi) automate this end-to-end — manual per-tenant configuration at scale is where this model usually breaks.

## See also

- [labels](labels.md) — `namespace` is typically the label that drives tenant routing
- [cardinality](cardinality.md) — per-tenant limit overrides live alongside global cardinality limits
