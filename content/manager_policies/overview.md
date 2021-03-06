---
layout: bt_wiki
title: Overview
category: Manager
draft: false
abstract: "A guide to Cloudify Policies"
weight: 1

types_yaml_link: http://www.getcloudify.org/spec/cloudify/3.3/types.yaml
diamond_plugin_ref: plugin-diamond.html
dsl_groups_spec: dsl-spec-groups.html
workflows_dsl_spec: dsl-spec-workflows.html
diamond_package_ref: https://github.com/BrightcoveOS/Diamond
---

{{% gsSummary %}}{{% /gsSummary %}}


# Overview

Policies provide a way of analyzing a stream of events that correspond to a group of nodes (and their instances).
The analysis process occurs in real time and enables triggering actions based on its outcome.

The stream of events for each node instance is read from a deployment specific RabbitMQ queue by the policy engine ([Riemann](http://riemann.io/)). It may come from any source but is typically generated by a monitoring agent installed on the node's host. (See: [Diamond Plugin]({{< relref "plugins/diamond.md" >}})).

The analysis is performed using [Riemann](http://riemann.io/). Cloudify starts a Riemann core for each deployment with its own Riemann configuration. The Riemann configuration is generated based on the blueprint's `groups` section that will be described later in this guide.

A part of the API that Cloudify provides on top of Riemann, allows activating policy triggers. Usually, that simply means executing a workflow in response to some state (e.g. The tomcat CPU usage was above 90% for the last five minutes).
That workflow may increase the number of node instances, for example, to handle an increased load.

The policies logic itself is written in [Clojure](http://clojure.org/) using Riemann's API. Cloudify adds a thin API layer on top of these API's.

# Using Policies

Policies are configured in the `groups` section of a blueprint.
An example:

{{< gsHighlight  yaml  >}}
groups:
  # arbitrary group name
  my_group:

    # all nodes this group applies to
    # in this case, assume node_vm and mongo_vm
    # were defined in the node_templates section.
    # All events that belong to node instances of
    # these nodes will be processed by all policies specified
    # within the group
    members: [node_vm, mongo_vm]

    # each group specifies a set of policies
    policies:

      # arbitrary policy name
      my_policy:

        # the policy type to use. Here we use one
        # of the built-in policy types that identifies
        # host failures based on a keep alive mechanism
        type: cloudify.policies.types.host_failure

        # policy specific configuration
        properties:
          # Not really a host_failure property, only serves as an example.
          some_prop: some_value

        # This section specifies what should be
        # triggered when the policy is "triggered"
        # (more than one trigger can be specified)
        triggers:
          # arbitrary trigger name
          execute_heal_workflow:

            # using the built-in 'execute_workflow' trigger
            type: cloudify.policies.triggers.execute_workflow

            # trigger specific configuration
            parameters:
              workflow: heal
              workflow_parameters:
                # the failing node instance id is exposed by the
                # host_failure policy on the event that triggered the
                # execute workflow trigger. Access to properties on
                # that event are as demonstrated below
                node_instance_id: { get_property: [SELF, node_id] }

{{< /gsHighlight >}}

# Built-in Policies

Cloudify comes with a number of built-in policies, which are declared in [`types.yaml`]({{< field "types_yaml_link" >}}) (which is usually imported either directly or indirectly via other imports).

Refer to [Built-in policies]({{< relref "manager_policies/built-in-policies.md" >}}) for more details.

# Writing a Custom Policy

Advanced users may wish to write custom policies.
To learn how to write a custom policy, refer to the [policies authoring guide]({{< relref "manager_policies/creating-policies.md" >}}).

# Auto Healing

Auto healing is the process of automatically identifying when a certain component of your system is failing and fixing the system state without any user intervention.

Cloudify comes with a built-in support for auto healing different components.

There are several things that need to be configured for auto healing to work. This guide will walk you through them and explain the mechanism's current limitations.

In short, the process involves:

* Configuring monitoring on compute nodes that will be subject to auto healing.
* Configuring the `groups` section with appropriate policy types and triggers.

## Monitoring

Add monitoring to compute nodes that require auto healing.

For the time being, there is a limitation requiring that only compute nodes can have auto heal configured for them. Otherwise, when a certain compute node instance fails, node instances that are contained in it are also likely to fail, which in turn can cause the heal workflow to be triggered multiple times.

We will show an example using the built-in [Diamond]({{< field "diamond_package_ref" >}}) support that comes with Cloudify. Please refer to the [Diamond Plugin]({{< relref "plugins/diamond.md" >}}) documentation for further details on configuring diamond based monitoring.

{{< gsHighlight  yaml  >}}
node_templates:
  some_vm:
    interfaces:
      cloudify.interfaces.monitoring:
        start:
          implementation: diamond.diamond_agent.tasks.add_collectors
          inputs:
            collectors_config:
              ExampleCollector: {}
        stop:
          implementation: diamond.diamond_agent.tasks.del_collectors
          inputs:
            collectors_config:
              ExampleCollector: {}
{{< /gsHighlight >}}

The `ExampleCollector` generates a single metric whose value is consistently `42`. We will use this collector as a "heart beat" collector combined with the `host_failure` policy that will be described below.

## [Workflow]({{< relref "blueprints/spec-workflows.md" >}}) Configuration

The `heal` workflow will execute a sequence of tasks that is similar in nature to calling the `uninstall` workflow and the `install` workflow thereafter. The main difference is that this workflow operates on a subset of node instances that are contained within the failing node instance's compute and on relationships these node instances have with any other node instances. In other words, this workflow reinstalls the whole compute that contains the failing node and handles all appropriate relationships between the node instances inside this compute and any other node instances.

## [Groups]({{< relref "blueprints/spec-groups.md" >}}) Configuration

After we have monitoring set up, it's time to plug things together.

This example contains several inline comments, so make sure you read them to have better understanding of
how things work.

{{< gsHighlight  yaml  >}}
groups:
  some_vm_group:
    # adding the some_vm node template that was previously configured
    members: [some_vm]
    policies:
      host_failure_policy:
        # using the 'host_failure' policy type
        type: cloudify.policies.host_failure

        # Name of the service we want to shortlist (using regular expressions) and
        # watch - every Diamond event has the service field set to some value.
        # In our case, the ExampleCollector sends events with this value set to "example".
        properties:
          service:
            - example

        triggers:
          heal_trigger:
            # using the 'execute_workflow' policy trigger
            type: cloudify.policies.triggers.execute_workflow
            parameters:
              # configuring this trigger to execute the heal workflow
              workflow: heal

              # The heal workflow will get
              # its parameters from the event that triggered
              # its execution
              workflow_parameters:
                # 'node_id' will be the node instance id
                # of the node that failed. In our case, it will be
                # something like 'some_vm_afd34'
                node_instance_id: { get_property: [SELF, node_id] }

                # Contextual information added by the triggering policy
                diagnose_value: { get_property: [SELF, diagnose] }
{{< /gsHighlight >}}

In this example, we have configured a group consisting of the `some_vm` node specified before.

Afterwards, we configured a single `host_failure` policy for this group.

The policy in the example has one `execute_workflow` policy trigger configured, which is mapped to execute the `heal` workflow.

## Limitations

{{% gsWarning title="Limitations" %}}
* At the moment, only compute nodes can have auto healing configured for them. Otherwise, when a certain compute node instance fails, node instances that are contained in it are also likely to fail, which in turn can cause the heal workflow to be triggered multiple times.
{{% /gsWarning %}}
