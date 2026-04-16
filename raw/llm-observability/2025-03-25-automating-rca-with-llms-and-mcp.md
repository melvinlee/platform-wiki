---
title: "Automating Root Cause Analysis with LLMs and MCP: From Golden Signals to Intelligent Response"
source: "https://medium.com/@pateljheel/automating-root-cause-analysis-with-llms-and-mcp-from-golden-signals-to-intelligent-response-b921e4d46829"
author:
  - "[[Jheel Patel]]"
published: 2025-03-25
created: 2026-04-16
description: "Automating Root Cause Analysis with LLMs and MCP: From Golden Signals to Intelligent Response Automation and Observability are the cornerstones of Site Reliability Engineering (SRE). SRE focuses on …"
tags:
  - "clippings"
---
Automation and Observability are the cornerstones of Site Reliability Engineering (SRE). SRE focuses on enabling engineers to concentrate more on development by leveraging automation and observability to reduce toil and enhance the reliability of applications and infrastructure.

SRE recommends monitoring four key metrics, latency, traffic, errors, and saturation, known as the Four Golden Signals, for user-facing applications. These signals provide direct insight into the health of your system, enabling you to focus on service quality monitoring rather than getting lost in indirect resource metrics.

When your Golden Signals indicate an anomaly, it’s a cue to drill down into detailed monitoring and logging to uncover the root cause. Although root cause analysis (RCA) has traditionally been a manual effort, the rise of automation and AI-driven observability tools is paving the way for more automated insights and faster RCA.

## Why Automating RCA Feels Like Herding Cats

There are several challenges to automating root cause analysis (RCA). First, applications are becoming increasingly distributed, making it expensive to analyze logging and monitoring data in real time. In most cases, such real-time analysis isn’t necessary when the system is operating normally, unless this monitoring is being done for security purposes.

Secondly, because both applications and infrastructure are distributed, the associated monitoring and logging data is also dispersed. This requires either custom integrations or centralized storage for data aggregation, both of which introduce significant complexity and operational challenges \[[1](https://www.causely.ai/blog/fools-gold-or-future-fixer-can-ai-powered-causality-crack-the-rca-code-for-cloud-native-applications), [2](https://drdroid.io/engineering-tools/applying-automated-root-cause-analysis-with-ai-and-machine-learning), [4](https://betterstack.com/community/guides/logging/log-aggregation/)\].

As of today, numerous solutions offer AI-driven intelligent analysis of logs and monitoring data. However, most of these require robust integrations and rely on predefined data aggregation flows, rather than dynamically adapting based on the specifics of each incident.

## How to Start Herding Those Cats

I was reading about the Model Context Protocol (MCP), which was recently open-sourced by Anthropic. I like to think of it as giving hands and eyes to large language models (LLMs). One of our greatest abilities as humans is working efficiently with unstructured data and navigating complex interfaces. MCP enables LLMs to do something similar — providing them with the context and tools to interact more effectively with their environment, making AI agents more capable than ever.

So, I was thinking, why not combine this with proactive monitoring of the Golden Signals? The idea is to set up alerts based solely on the Four Golden Signals and configure a webhook to receive those alerts. Once an alert or incident is triggered, an LLM agent can process the alert payload and, using the appropriate MCP-based tools from a predefined list, fetch the necessary monitoring and logging data from various sources. Based on this data, the agent can then evaluate the root cause and even provide suggestions to resolve the issue.

Before proceeding, I recommend reviewing the MCP Quickstart documentation \[[3](https://modelcontextprotocol.io/introduction)\].

## Benefits of Using LLMs with MCP for Automated RCA

- No need for complex data parsers, unlike traditional rule-based systems, LLMs can work directly with unstructured data from various sources.
- Better handling of edge cases, both in interpreting input, invoking tools and processing results.
- Efficient use of monitoring and logging data — only the necessary data is retrieved, and only when it’s needed.
- Eliminates the need for manual interaction with complex interfaces.
- Easily incorporates business logic and operational practices using customizable prompt templates.
- Shifts RCA automation from an imperative approach (step-by-step logic) to a more declarative one, focused on outcomes, not instructions.
- Allows dynamic execution of data aggregators or collectors based on the specifics of an incident, rather than relying on prebuilt, static logic.

## A Simple Example

To test my idea, I created a very simple web application hosted on Google Cloud. It consists of a Regional External HTTP/S Load Balancer, a VM instance hosting a simple REST API in an Unmanaged Instance Group, and a Cloud SQL MySQL database. The API is straightforward: it accepts a file, uploads it to a GCS bucket, and performs a dummy operation on the Cloud SQL instance. By default, the VM’s `stderr` and `stdout` streams are sent to Cloud Logging.

To monitor application health, I selected two Golden Signals on the external Load Balancer (LB) for demonstration purposes: total latency and 5XX errors. I created two alert policies that trigger when either high latency is detected or the rate of 5XX errors exceeds a defined threshold. When an alert is triggered, the payload is sent to a webhook that invokes my MCP Client, powered by Claude LLM, which has access to my MCP server.

## Get Jheel Patel’s stories in your inbox

Join Medium for free to get updates from this writer.

I implemented one MCP server with two tools:

1. **Prometheus Query Tool**: This tool accepts a list of PromQL queries along with their intended purposes. It retrieves the required monitoring data from Google Cloud Monitoring.
2. **Logging Query Tool**: This tool accepts a Cloud Logging query to retrieve relevant error logs from Google Cloud Logging.

The LLM prompt was carefully crafted to guide the agent to use these tools judiciously, only when necessary, to optimize token usage and minimize API calls to Google Cloud services. Once it has enough data the LLM agent generates the analysis, possible root cause and suggestions to resolve it.

Overall, the setup looks like this in the following diagram.

![](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*ML7wWSXAho0DsZewRMqEAw.png)

Fig. 1. Application and Automated RCA System

This is a very simple example that might not necessarily require an automated RCA system, but that’s not the point of this demonstration.

Here’s a video demonstrating the PoC.

## Key Considerations When Implementing Your Own MCP Tools

- **Implement Robust Exception Handling:** Include clear and descriptive error messages in your tools. This allows the MCP Client to retry intelligently by modifying its arguments based on the specific errors encountered.
- **Balance Prompt vs. Tool Complexity:** There’s a tradeoff between the complexity of your prompt and that of your tools. A more expressive prompt can allow for simpler tool implementations, but it may significantly increase LLM usage and associated costs.
- **Filter Data Before Sending to the LLM:** Apply filtering or pre-processing to the data collected by MCP tools. Reducing the volume of unnecessary or irrelevant data can help lower LLM token usage and improve performance.
- **Use Expressive Docstrings for Tools:** Provide clear, detailed, and purpose-driven docstrings for each tool. This improves the LLM agent’s understanding of how to use them effectively, reducing errors and unnecessary retries.

## How Can We Extend It?

Some interesting ways to extend this solution are:

1. Develop tools to remediate the root cause of issues that can be resolved automatically without further risk. Automating well-understood fixes reduces downtime and increases efficiency.
2. Build tools to integrate with your ITSM (IT Service Management) platform to manage incidents. This improves service management by enabling automatic incident creation, updates, and resolution tracking.
3. Experiment with different prompt templates. Changing the structure or wording of prompts can improve the accuracy, clarity, and effectiveness of the solution and the tool use.

The complete code is hosted [here](https://github.com/scienceto/mcp-rca). Thanks for taking the time to read my blog.