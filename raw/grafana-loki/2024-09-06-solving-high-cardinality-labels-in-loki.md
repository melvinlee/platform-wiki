---
title: "Solving high cardinality labels in Loki - charm"
source: "https://discourse.charmhub.io/t/solving-high-cardinality-labels-in-loki/15426"
author:
  - "[[jose]]"
published: 2024-09-06
created: 2026-04-15
description: "IntroductionContrary to what happens with indexes in SQL databases like MySQL or PostgreSQL, high cardinality is something to avoid when talking about labels in Loki. In SQL databases, we look for indexes to have high …"
tags:
  - "clippings"
---
## Introduction

Contrary to what happens with indexes in SQL databases like MySQL or PostgreSQL, high cardinality is something to avoid when talking about labels in Loki.

In SQL databases, we look for indexes to have high cardinality because this improves query efficiency by allowing the index to quickly filter the results. So, while in Loki this can be a problem, in SQL it can be an advantage for optimising data access.

Having high cardinality in labels is problematic because it means there are too many unique values for a specific label, significantly increasing the number of streams that need to be stored and managed. This can lead to higher memory usage, impact query performance, and cause storage overload and make it difficult to scale the system efficiently. In short, high cardinality can overwhelm system resources and slow down its overall operation.

## How Loki uses labels

According to [Loki documentation](https://grafana.com/docs/loki/latest/get-started/labels/#how-loki-uses-labels):

> Labels in Loki perform a very important task: They define a stream. More specifically, the combination of every label **key** and **value** defines the **stream**. If just one label value changes, this creates a new stream.
> 
> If you are familiar with Prometheus, the term used there is series; however, Prometheus has an additional dimension: metric name. Loki simplifies this in that there are no metric names, just labels, and we decided to use streams instead of series.

## When we can face a high cardinality situation?

[@FaQ](https://discourse.charmhub.io/u/faq) [was very clear when he explained his situation](https://github.com/canonical/loki-k8s-operator/issues/439):

> *"When using Loki to ingest logs from an Openstack cloud, the inclusion of the `filename` label for each log line can potentially lead to a huge cardinality.*
> 
> *Openstack uses libvirt to spawn VMs, and libvirt creates one log file for each VM launched (`/var/log/libvirt/qemu/$DOMAIN_NAME.log`). Some clouds used as build farms, can spawn thousands of VMs in the course of a few days, each with it’s own value for the `filename` label."*

Also we should avoid labels that include `process_id`, `thread_id` or Kubernetes pod names.

## Is it possible to address this situation in Loki + Grafana agent charms?

The short answer is: **Yes**, we need `structured_metadata`

The long answer is still **Yes**, but the devil is in the details

## Loki

[Since Loki 2.9.4](https://grafana.com/docs/loki/latest/release-notes/v2-9/#features-and-enhancements) the [`structured_metadata` feature](https://grafana.com/docs/loki/latest/get-started/labels/structured-metadata/#what-is-structured-metadata) is GA.

> Structured metadata is a way to attach metadata to logs **without indexing them** or including them in the log line content itself. Examples of useful metadata are kubernetes pod names, process ID’s, or any other label that is often used in queries but has high cardinality and is expensive to extract at query time.

So it feels like it’s what we need in our OpenStack situation described by [@FaQ](https://discourse.charmhub.io/u/faq)!

In order to enable that feature in Loki we need to do two things:

1. Enable `allow_structured_metadata` in the config file:
	```yaml
	limits_config:
	  allow_structured_metadata: true
	```
2. Make sure the `schema_config` we are using is at least `v13`:
	```yaml
	schema_config:
	  configs:
	  - from: '2024-09-03'
	    index:
	      period: 24h
	      prefix: index_
	    object_store: filesystem
	    schema: v13
	    store: tsdb
	```

We have a [WIP PR](https://github.com/canonical/loki-k8s-operator/pull/445) for this.

## Grafana Agent

Once `structure_metadata` is enabled in Loki, we need to scrape and send logs including the high cardinality label as `structured_metadata` instead of a regular label.

In order to do that, we need to modify the grafana-agent `scrape_config` definition with something like this:

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
  static_configs:
  - labels:
      __path__: /var/log/**/*log
      instance: juju-ccf9c7-7.lxd
      job: varlog_scraper
      juju_application: agent
      juju_charm: grafana-agent
      juju_model: apps
      juju_model_uuid: c157bd5e-8306-40a9-8307-b00c63ccf9c7
      juju_unit: agent/13
    targets:
    - localhost
```

The important part of this piece of config is `pipeline_stages`. Let’s go one by one:

1. `drop` stage:
	- This stage filters out any log entries that match the regular expression `.*file is a directory.*`. If the log message contains this pattern, it is discarded, preventing unnecessary logs from being sent to Loki. (Note that this stage is already in grafana-agent charm.)
2. `structured_metadata` stage:
	- In this stage, the `filename` label is added to the structured metadata, associating the log entry with the corresponding log file name. This allows the filename to be stored in the log’s metadata without indexing it as a label, helping reduce cardinality.
3. `labeldrop` stage:
	- The `labeldrop` step removes the `filename` label after it has been processed and added to structured metadata. This ensures that the label doesn’t contribute to high cardinality by being indexed.

Each step processes log entries sequentially, performing specific actions to control what information is kept, how it’s stored, and which parts are discarded.

## How do we see this in Grafana?

As we saw in `grafana-agent` configuration we scrape all the files that matches the regex `/var/log/**/*log`, for instance `/var/log/pepe.log`.

So in order to search for logs with the `job="varlog_scraper"` label, and the `filename` metadata `/var/log/pepe.log` we need to execute this `LogQL` query:

```
{job="varlog_scraper"} | filename="/var/log/pepe.log"
```

to get this in Grafana:

[![image](https://discourse-charmhub-io.s3.eu-west-2.amazonaws.com/optimized/2X/a/a4e70b8015b8ca6618c71a8f57aecdc4f3ae541a_2_690x766.png)](https://discourse-charmhub-io.s3.eu-west-2.amazonaws.com/original/2X/a/a4e70b8015b8ca6618c71a8f57aecdc4f3ae541a.png "image")

Some important comments should be made about this:

Although `filename` key seems to be part of the regular (indexed) labels it is not, because we are not including it inside the `{ }`, we use instead `|`. At this point Grafana UI is not clear enough to let us know at glance which labels are indexed and which ones are part of `structured_metadata`.

We can verify this by trying to use `filename` as a regular label by executing:

```
{job="varlog_scraper", filename="/var/log/pepe.log"}
```

The result of that query is an empty set:

## Final thoughts

The decision to move the `filename` label into `structured_metadata` rather than indexing it is crucial for reducing cardinality in Loki. As we said before indexing labels like `filename`, would lead to an overwhelming number of streams. This not only affects Loki’s storage efficiency but also impacts query performance, making the system harder to scale. By storing `filename` as metadata, we retain the ability to filter logs without unnecessarily inflating the cardinality.

Providing Juju administrators flexibility in defining `scrape_configs` is important, but it’s also key to follow best practices to avoid performance issues. Indexing highly variable labels, like filenames, can increase cardinality and strain resources. Using structured metadata for such labels allows for effective log filtering while maintaining a scalable, efficient Loki deployment. This ensures a responsive system that balances flexibility with performance.