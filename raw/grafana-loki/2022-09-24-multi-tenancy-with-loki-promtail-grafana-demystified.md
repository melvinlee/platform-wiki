---
title: "Multi-tenancy with Loki, Promtail, and Grafana demystified"
source: "https://medium.com/otomi-platform/multi-tenancy-with-loki-promtail-and-grafana-demystified-e93a2a314473"
author:
  - "[[Sander Rodenhuis]]"
published: 2022-09-24
created: 2026-04-15
description: "Multi-tenancy with Loki, Promtail, and Grafana demystified Learn how to create an enterprise-grade multi-tenant logging setup using Loki, Grafana, and Promtail. Last week we encountered an issue with …"
tags:
  - "clippings"
---
## [Akamai App Platform](https://medium.com/otomi-platform?source=post_page---publication_nav-a8cb98f3f0c3-e93a2a314473---------------------------------------)

[![Akamai App Platform](https://miro.medium.com/v2/resize:fill:76:76/1*6URA98gG4XLRx_B8uLJxIA.png)](https://medium.com/otomi-platform?source=post_page---post_publication_sidebar-a8cb98f3f0c3-e93a2a314473---------------------------------------)

Technical articles from the creators of Akamai App Platform

![](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*m3lvdr7YGtJ17Cu0MM_fwQ.png)

Learn how to create an enterprise-grade multi-tenant logging setup using Loki, Grafana, and Promtail.

Last week we encountered an issue with the Loki multi-tenancy feature in [otomi](https://github.com/redkubes/otomi-core). Although the otomi-core code is very well structured, some research was required. During my research, I noticed that there is a lack of good articles out there describing how to set up Loki multi-tenancy. In this article, I will explain how multi-tenancy with Loki, Promtail, and Grfana can be done using otomi-core as a reference architecture.

But to answer your first question: “Did you fix the issue?”. Yes, I did. After debugging for a couple of days I first thought we had to start again from scratch, but eventually (as this usually goes when working with multiple solutions glued together) the issue was caused by a change in the Grafana chart on how to configure the password in the `additionalDataSources`. More on this later.

Let me start by describing the high-level architecture of the setup.

## Architecture

The following diagram shows the architecture of the setup.

![](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*VoI5kN_4QdVFo2Sx26AInw.png)

Loki multi-tenant architecture

How it works:

1. Users log in with Oauth2 (using an Identity Broker or Provider). Only users belonging to a tenant Group are allowed to get access to the Grafana instance of the tenant. The normalized JWT token is passed on to Grafana.
2. Each tenant namespace runs its own Grafana, but users can not create/update data sources in Grafana by themselves. Data sources are defined by the platform admin and can not be changed. The Datasource for Loki points to a reverse proxy, running as a sidecar next to Loki in the same Pod.
3. The reverse proxy uses a secret that contains the user/password combination for the tenant and the tenant name. When authentication is successful, the tenant name will be used in the *X-Scope-OrgID* header and passed on to Loki.
4. Loki is configured with `auth_enabled=true`
5. Promtail is configured with a default `tenant_id=admins` and a snippet is used to configure the pipeline stages to ‘mark’ all logs belonging to a specific tenant with the name of the tenant.

Seems easy right? Let’s dive in a little deeper. The ingress setup using an oauth2/Istio/nginx/Keycloak is out of scope here.

## Promtail

I’m not going into all the possible Promtail configuration options but will focus only on the relevant configuration for the multi-tenant setup.

1. Configure the `clients` section in the Promtail chart values.yaml:
```rb
config:
  clients:
    - url: http://loki.monitoring:3100/loki/api/v1/push
      tenant_id: admins
```

Notice that we are not providing a username and password. Promtail connects to the Loki service without authentication. Promtail and Loki are running in an isolated (monitoring) namespace that is only accessible for admins.

## Get Sander Rodenhuis’s stories in your inbox

Join Medium for free to get updates from this writer.

2\. Add a snippet to configure the `pipelineStages`:

```rb
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

A pipeline (source can be found [here](https://grafana.com/docs/loki/latest/clients/promtail/pipelines/)) is used to transform a single log line, its labels, and its timestamp. A pipeline is comprised of a set of stages. There are 4 types of stages:

1. Parsing stages parse the current log line and extract data out of it. The extracted data is then available for use by other stages.
2. Transform stages transform extracted data from previous stages.
3. Action stages take extracted data from previous stages and do something with them.
4. Filtering stages optionally apply a subset of stages or drop entries based on some condition.

In summary: All logs that match the namespace name of the tenant are marked as \`owned\` by the tenant. If the tenant name matches the *X-Scope-OrgID* in the HTTP header sent to Loki, Loki will only return logs of that tenant.

## Loki

To use Loki in multi-tenant mode, you’ll need to do 2 things:

1. Configure Loki with auth\_enabled:
```rb
config:
  auth_enabled: true
```

2\. Add a reverse proxy to handle the authentication and add the `X-Scope-OrgID` HTTP header for Loki. I stumbled upon a handy solution and forked it: [https://github.com/redkubes/loki-multi-tenant-proxy](https://github.com/redkubes/loki-multi-tenant-proxy) but you can also create one yourself. The reverse proxy is added as a side-car container in the Loki Pod.

```rb
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
        protocol: TCP
    resources:
      limits:
        cpu: 250m
        memory: 200Mi
      requests:
        cpu: 50m
        memory: 40Mi
    volumeMounts:
      - name: reverse-proxy-auth-config
        mountPath: /etc/reverse-proxy-confextraVolumes:
  - name: reverse-proxy-auth-config
    secret:
      secretName: reverse-proxy-auth-configextraPorts:
  - port: 3101
    protocol: TCP
    name: http
    targetPort: http
```

3\. Create a secret containing an authn.yaml file with the usernames, passwords, and tenant names for all the tenants. This secret will be mounted to the reverse proxy.

```rb
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
stringData:
  authn.yaml: |
    - username: admin
      password: password
      orgid: admins
    - username: tentant-1
      password: password
      orgid: tenant-1
    - username: tenant-2
      password: password
      orgid: tenant-2
```

## Grafana

To enable tenants to query their logs, add an additional data source for to the tenants' Grafana instance:

```rb
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

Note that this needs to be done for each tenant’s Grafana instance! Read more on configuring data sources [here](https://grafana.com/docs/grafana/latest/administration/provisioning/#datasources).

Now, this is what caused my issue: the usage of the `basicAuthPassword` has changed in one of the latest Grafana versions and is only supported as a `secureJsonData` property. A tool that automatically checks values changes during upgrades would be nice;-)

In loki, users will see a provisioned data source that they can use (but can not edit):

![](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*81_x9uSdPdLjH6d1JdQ0bw.png)

## Wrapping up

Note that the setup I described here is still depending on a good authentication and authorization mechanism to make sure only tenant users are able to access the tenants' own Grafana. When this is done, users can only access the logs of the tenant to where they belong to. Check out the [otomi-core](https://github.com/redkubes/otomi-core) repo if you're looking for a good reverence on how to set up an advanced multi-tenant ingress architecture using oauth2, Istio, Nginx ingress controller and Keycloak

Creating a multi-tenant setup for Loki doesn’t seem to be really complex after you figured out how to do it. But remember that the essence here is not on how to set this up, but on how to operate it. If you have multiple tenants that need to be onboarded each day, then you might be spending a lot of time on this.

In Otomi you can create a new tenant with just 2 clicks: Click on create Team, provide a name and click on submit. Otomi will then create a namespace for the tenant, install Grafana in it, create a tenant password (encrypted using SOPS), add the tenant to the authentication proxy, add the Loki data source with the tenant information to Grafana and add the tenant to the Promtail pipeline stages. Everything is completely automated and all tenants’ configurations are stored in Git.

Hope you like our effort on the [Otomi](https://github.com/redkubes/otomi-core) project and support us by starring. Thanks!

If you have any questions, you can contact me on L [inkedIn](https://www.linkedin.com/in/srodenhuis/) or [Twitter](https://twitter.com/SanderRodenhuis).[Last published Jan 5, 2024](https://medium.com/otomi-platform/helmfile-and-argocd-better-together-f8d4587263ff?source=post_page---post_publication_info--e93a2a314473---------------------------------------)

Technical articles from the creators of Akamai App Platform

## Responses (1)

Write a response[What are your thoughts?](https://medium.com/m/signin?operation=register&redirect=https%3A%2F%2Fmedium.com%2Fotomi-platform%2Fmulti-tenancy-with-loki-promtail-and-grafana-demystified-e93a2a314473&source=---post_responses--e93a2a314473---------------------respond_sidebar------------------)

==mysecret==

```c
reverse-proxy-auth-config
```