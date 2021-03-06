---
layout: bt_wiki
title: Overview
category: Manager
draft: false
weight: 1

terminology_link: reference-terminology.html
---
{{% gsSummary %}}{{% /gsSummary %}}


Cloudify Manager comprises the following main parts:

* [Manager](#the-manager)
* [Agents](#the-agents) (Unless you're running in agent-less mode)
* [Web UI](#the-web-ui) (only in Commercial Edition)

# The Manager

Cloudify Manager comprises of Cloudify's code and a set of Open-Source applications. An elaborate explanation on these applications is provided [here]({{< relref "manager_architecture/components.md" >}}).

The Manager's architecture is designed in such a way to provide support for all potential operational flows you might require when managing your applications such as:

* Event Stream Processing
* Secured Requests
* Metrics Queuing, Aggregation and Analysis
* Logs/Events Queuing, Aggregation and Analysis
* Manual or Automated Task execution and Queuing based on live streams of events or aggregated data.
* Interaction with Cloudify's Agents for executing tasks on application hosts and maintaining them.

You can also communicate with the Manager via the CLI, which uses Cloudify's REST client module to interact with the Cloudify REST service.

All requests are served via a proxy.

# The Agents

Cloudify's Agents are entities designed to execute tasks on application hosts. They're able to listen to task queues and execute tasks when required.

The agents are designed to execute tasks using [Cloudify specific Plugins]({{< relref "plugins/overview.md" >}}).

In the background, the same agents used on the hosts are also used in the Manager but in a different context. For instance, each deployment has two agents, one of which talks to IaaS APIs to deploy resources.

{{% gsNote title="Note" %}}
Note that Cloudify can run in "Agentless" mode which means that agents can use certain plugins to manage hosts without the agents being installed on them. A user can decide which server nodes will have agents installed on them by stating the choice in the blueprint.
{{% /gsNote %}}

More on agents [here]({{< relref "agents/overview.md" >}}).

# The Web UI

Cloudify's Commercial Edition features a Web UI. The Web UI provides most of the features the CLI provides and more. For now, Cloudify's Web UI is deployed alongside the manager. We're working on modularization and decoupling so that it can be deployed anywhere.
