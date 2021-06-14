---
layout: post
title:  "Managing Azure Policies with Infrastructure as Code principles: configuring diagnostic logs (Part 1)"
date:   2021-06-09
description: In this post we’ll talk about one of the most important part of Azure Governance – Azure Policy. I will describe how to manage policies using Infrastructure as Code (IaC) principles and pipelines in Azure DevOps.
categories:
  - Azure
tags:
  - Azure
  - Azure Governance
  - Azure Policy
  - Log Analytics
---

{:refdef: style="text-align: center;"}
![Title image]({{ site.baseimg }}\assets\img\2021\2021-06-09\title.jpg)
{: refdef}


<p class="intro"><span class="dropcap">I</span>n this post we’ll talk about one of the most important part of Azure Governance – Azure Policy. In this part there will be overview of Azure policies together with explaining how to use it for configuring Diagnostic Logs for Azure resources</p>

### Azure governance components overview

Let’s briefly recall what Azure Policy is and why they are important. As more and more workloads are migrated or built natively in Azure, it becomes crucial how to effectively manage and control multiple subscriptions. That’s very important for enterprises as usual approach is to have dedicated team offering managed pre-configured subscriptions to internal customers, for example, departments or companies inside a big holding. Ensuring that compliance and security level meets security requirement of a company is a complex task for CloudSecOps team after subscription is pre-built and offered to a customer.

Azure policy is an automated way to check if resources are met certain requirements. If resources are not compliant with policies, it trigger alert or can perform automatic remediation. Policies are described as ARM templates. Each subscriptions have bunch of built-in policies that can satisfy most of requirement around compliance. Also Microsoft provides very detailed documentation about policies in general and [built-in ones](https://docs.microsoft.com/en-us/azure/governance/policy/samples/built-in-policies). Indeed, it’s possible to develop your own custom policy, although writing and debugging ARM templates might be complex task. Policies stored as so-called definitions and should be assigned to subscription first to have effect. Policies could be grouped into initiatives, which is a logical set of policies that requires only to be assigned once.  

Before [Azure Management group](https://docs.microsoft.com/en-us/azure/governance/management-groups/overview) feature was introduced, each policy could only be defined and assigned at subscription level. This obviously involved a lot of administrative burden if you must manage dozens of subscriptions and custom policies. Management group (MG) is a logical container for subscriptions and child MG. Now it’s possible to define policies (and also custom roles and blueprints) and assign them at MG level so they are inherited to all subscriptions below. Management groups could be organized in hierarchical tree up to 6 level deep. This indeed makes the whole governance much more efficient. After MG feature is activated, root management group is automatically created with random GUID as an ID. All child MG can have more descriptive names.

![Azure Management group structure](\assets\img\2021\2021-06-09\MG-structure.png)

### DeployIfNotExists and managed identity

Here I need to tell few words about DeployIfNotExists assignment as it is important to understand later. Policy can deploy resources using DeployIfNotExists . It deploys resource if it's missing as the name suggests. One important aspect however is, by default Azure Policy engine doesn't have access to resources in a subscription. This makes it impossible to automate deployment without either manually triggering [remediation task](https://docs.microsoft.com/en-us/azure/governance/policy/how-to/remediate-resources#create-a-remediation-task) or [manually configuring managed identity for policy assignment first](https://docs.microsoft.com/en-us/azure/governance/policy/how-to/remediate-resources#manually-configure-the-managed-identity).

![Managed identity for policy assignment](\assets\img\2021\2021-06-09\policy-MSI.png)

This makes it's more difficult as you might want to assign single policy or initiative with `DeployIfNotExists` multiple times for subscriptions or management groups and in every case have to assign Azure IAM roles again. Later I'll show how to automate this task. This limitation also has other side effects: 
- Since policy assignment name is limited only to 24 characters, service principals corresponding to managed identities created during policy assignment might have cryptic short names
- After removing policy assignment role assignment will remain as *'Identity not found'* entry

!['Identity not found' entry](\assets\img\2021\2021-06-09\Unknown-identity.png)

There is [opened Azure feedback post](https://feedback.azure.com/forums/915958-azure-governance/suggestions/39977704-deployifnotexists-policy-add-user-assigned-manag) suggesting to enable use of user assigned managed identities for policies, in 'Planned' state at the time of the writing (June 2021). Also, if I tried to make a trick specifying Id of existing system managed identity for policy assignment, but Azure created new MSI anyway.

### Azure Policy to manage Diagnostic Log for resources

When designing governance for multiple subscriptions central logging for  audit, security incident response and other reasons becomes crucial too. Microsoft provides Log Analytics (LA) as a managed service to store logs and metrics. It can then provide stored data to services like SIEM, however, some simple incident management could be implemented with built-in alerting feature of Log Analytics. Azure resource audit logs could be [configured for forwarding to LA workspace](https://docs.microsoft.com/en-us/azure/azure-monitor/essentials/diagnostic-settings), however configuring central storage for diagnostic logs of multiple Azure resources could be challenging when resources are not deployed centrally. This is often the case since CloudSecOps team usually provides subscription for customers who deploy workloads and responsible for the infrastructure they build.

![Central Logging with Log Analytics](\assets\img\2021\2021-06-09\LA-Logging.png)

Azure policy is a perfect solution for such task. Diagnostic logs is a resource property that could be deployed through ARM using [DeployIfNotExists](https://docs.microsoft.com/en-us/azure/governance/policy/concepts/effects#deployifnotexists) definition. 

Tao Yang did a brilliant job developing Azure policy initiative that enables diagnostic log forwarding to Log Analytics workspace.

[*Over the last 3-4 days, I was up until 2am each day working on this template due to the lack of documentation. I have PERSONALLY tested all 47 resources included in this template by creating resources and monitoring the subsequent deployments initiated by the Azure Policy engine. In the end, this 5000+ line gigantic template is born.*](https://blog.tyang.org/2018/11/19/configuring-azure-resources-diagnostic-log-settings-using-azure-policy/)

So all Kudos definitely go to him. I modified his version of policy initiative to add more resources like Azure Bastion and, what is more important, now it could be defined and assigned at management group level. This policy initiative is published in my GitHub.
Let's us briefly take a look at the initiative's ARM template as that is important for our next topic related to implementing IaC.
1. template contains complete policies defined as array of objects with type "Microsoft.Authorization/policyDefinitions"
2. initiative itself is defined in the last section with type "Microsoft.Authorization/policySetDefinitions"
3. initiative has array attribute `policyDefinitions` that contains full path to each policy definition. As we publish initiative to a management group, all `policyDefinitionId` properties inside the array should point to MG we're about publish policies to
4. To make it easier, I added special parameter ManagementGroupId inside initiative to provide management group id in form `"/providers/microsoft.management/managementGroups/ManagementGroupName"`


```json
    "parameters": {
    "ManagementGroupId": {
      "type": "String",
      "metadata": {
        "description": "..."
      }
    }
  }
```

After policy set is deployed and assigned, Azure policy engine starts to report on compliance issues and create remediation task for diagnostic logs. 

![Assigned policy set](\assets\img\2021\2021-06-09\LA-portal.png)

Remediation tasks could be tracked in Azure audit logs.

![Remediation in Azure audit logs](\assets\img\2021\2021-06-09\Policy-Logs.png)

**Note, automatic remediation will occur only to resources created after policy set was assigned. This is known issue and here is [topic from 2019 in Azure feedback forum for this](https://feedback.azure.com/forums/915958-azure-governance/suggestions/35897443-allow-automatic-remediation-of-deployifnotexists-t).**


#### Policy code

**Policy initiative could be found in [Azure Policies repo of my GitHub](https://github.com/Astashin/Azure-Governance/blob/main/Policies/diagnosticlogs-to-loganalytics/policy.json).** 

OK, that was a short one. I decided to split the post into two parts. In the next one I'll explain how to configure pipeline in Azure DevOps for automatic deployment of this and any other policy initiative to Azure.



