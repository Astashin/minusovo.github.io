---
layout: post
title:  "Difference between various Event Grid resources in Azure"
date:   2020-04-16
description: Azure has three separate servies all having Evend Grid in names
categories:
  - Azure
tags:
  - Azure
  - Azure Event Grid
---

<p class="intro"><span class="dropcap">A</span>zure has three separate resources all having Evend Grid in names. However, they have some big differences when it comes how to work with them.</p>

Azure has three separate servies all having Evend Grid in names. Let's find out the difference between them. 

#### 1. Azure Event Grid 
Azure Event Grid is a service to send events from pre-defined supported sources (other resources)

![](\assets\img\2020\2020-04-16\EventGrid.png)

When configured no resource with type Event Grid is created. Instead, this service will be part of source resource. For example, if you configure subscription for events coming to Event Hub, then `Microsoft.EventGrid resource` is created part of its object:

{%- highlight json-doc -%}
{
/subscriptions/<subscription_id>/resourceGroups/resourcegroup/providers/Microsoft.EventHub/namespaces/eventhub/providers/Microsoft.EventGrid/eventSubscriptions/subscription1
}
{%- endhighlight -%}
Event grid is not  displayed in Azure Portal UI as separate resource in a resource group.

#### 2. Event Grid Topics.
This is a separate resource that should be placed to a resource group the same way as any other Azure resources. EG Topic allows to use custom events to be routed to event handlers in Azure (Logic Apps, Functions, etc) or custom applications (using web hooks). 
Working with custom topics good shown in [this video](https://www.youtube.com/watch?v=gOpzNlyj3tY). 
Simply you send HTTP POST request to EG Topic which has specific endpoint URL with event as body in JSON format, additionally specifying SAS key in header (`“aeg-sas-key”`). This them triggers event forwarding to topic subscriber (event handler).

#### 3. Event Grid Domains.
[This how Microsoft describes it:](https://docs.microsoft.com/en-us/azure/event-grid/event-domains):
*An event domain is a management tool for large numbers of Event Grid topics related to the same application. You can think of it as a meta-topic that can have thousands of individual topics. Event domains make available to you the same architecture used by Azure services (like Storage and IoT Hub) to publish their events. They allow you to publish events to thousands of topics. Domains also give you authorization and authentication control over each topic so you can partition your tenants.*

Publishing events is similar to how it’s done for an Event Grid topic:

*When you create an event domain, you're given a publishing endpoint similar to if you had created a topic in Event Grid. To publish events to any topic in an Event Domain, push the events to the domain's endpoint the [same way you would for a custom topic](https://docs.microsoft.com/en-us/azure/event-grid/post-to-custom-topic). The only difference is that you must specify the topic you'd like the event to be delivered to.*

Topic is specified adding additional attribute to event body in JSON: `"topic": "SomeTopicInDomain"`.
You need to use RBAC to restrict who can subscribe to which topic. However, if event handlers are web hooks, there is no need to assign roles. Event will be sent to handler as HTTP Post request without authentication headers. 

 