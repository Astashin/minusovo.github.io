---
layout: post
title:  "Single Azure AD tenant for large enterprises, part 5: Sync between Azure AD tenants"
date:   2021-05-06
description: In this part of our series we will talk about how to establish "trusts" between Azure AD tenants
categories:
  - Azure AD
tags:
  - Azure AD
  - Logic Apps
  - Azure AD B2B
---

<p class="intro"><span class="dropcap">W</span>hen designing single tenant architecture for Azure AD for enterprise consisting of multiple companies, it is not uncommon that some of them already have users and data in their own tenant. How to proceed in such case?</p>

Microsoft does not provide trusts concept across Azure AD tenants similar way like in Active Directory. Before sign-in to a tenant, user identity should already exist there. Collaboration across tenants is implemented as Azure AD B2B feature in form of inviting external users from other tenants. After invitation redemption users can sign-in to services in destination tenant, however primary authentication is performed in their source tenant. There is no built-in mechanism to synchronize users across tenants besides bulk invitation and scripting.

Microsoft wrote excellent whitepaper describing all possible scenarios for B2B collaboration. Unfortunately, it is somehow hidden from search and it's not easy to get it without knowing direct link: [Multi-tenant user collaboration patterns in Azure Active Directory](https://aka.ms/Multi-tenant-users). Among other topics, it describes pros and cons choosing global tenant versus using multiple. But let us imagine, we decided to have central tenant architecture but still must keep other tenants and users inside. What is missing in this approach is a way to sync users from all “branch” tenants to single one. The whitepaper basically suggests you implement custom sync engine to make it possible:

*With [Delta Query](https://docs.microsoft.com/en-us/graph/delta-query-overview), tenant admins can deploy a scripted “pull” process to automate discovery and provisioning of identities to support resource access. This process checks the home tenant for new users and uses the B2B APIs to provision those users as invited users in the resource tenant.*

![Azure AD B2B sync Engine](\assets\img\2021\2021-05-06\B2BsyncEngine.png)

Perfect, but who can help us with the sync engine?  When we asked this question to Microsoft Premier Services, they it is possible to get an as additional service. Alternative way would be using third party tool or implementing your own custom automation. Without considering third party products, let's see which options can we quickly find.

#### Automating guest invitation using MS MIM

When looking for possible options automating B2B invitation, you can find one that uses Microsoft Identity Manager(MIM) 2016. It is described in details in the following blog series:
* [Automating Azure AD B2B Guest Invitations using Microsoft Identity Manager](https://blog.darrenjrobinson.com/automating-azure-ad-b2b-guest-invitations-using-microsoft-identity-manager)
* [Azure AD B2B Guest Invitations Microsoft Identity Manager Management Agent](https://blog.darrenjrobinson.com/updated-azure-ad-b2b-guest-invitations-microsoft-identity-manager-management-agent)

![Automating Azure AD B2B Guest Invitations using Microsoft Identity Manager](https://i0.wp.com/blog.darrenjrobinson.com/wp-content/uploads/2018/08/B2B-Invitation-MA-640px.png?w=640&ssl=1)

MIM is an old on-premises identity management product from Microsoft, which I mentioned already about in part 1 of these blog series. This solution is relatively easy to implement especially if you have previous experience working with MIM or MIIS and PowerShell, but it has some cons:
* MIM must be installed to VM, requiring to additionally perform tasks to monitor OS, audit, patching
* No way for clustering and provide fault tolerance at application level
* Attribute synchronization must be additionally implemented
* MIM requires license

#### Automating guest invitation and attribute synchronization using custom Azure Logic Apps

In this article I will share my own automation solution for synchronizing B2B users across different tenants. As name of this chapter suggests, it is based on Azure Logic Apps, althpugh this might seem like a strange choice. Logic Apps is best for small workflows and current automation is quite big one. This is true and it all started as a much smaller workflow than it became later. Here are pros and cons using Logic Apps for such complex automation.

**Pros:**
* Managed serverless service: all operations and audit is performed by Microsoft
* Does not require to write high-level code or scripts for building workflows (but for someone it should be disadvantage)
* Native use of managed identities for privileged actions in Azure AD
* It's possible to see output of each step for troubleshooting
* Native support of asynchronous run for loops

**Cons:**
* Writing automation in Logic Apps requires additional skill as it's very different from classical programming languages
* User experience writing and debugging workflows is sometimes slow and not intuitive
* Running big workflows too often can generate a lot of costs
* It's difficult to share the coding experience as the code is a complex JSON file (although steps can contain comments)
* Logic Apps generally runs slower than low-level code (running in Azure Function)

New generation of Logic Apps (currently in preview) allows to build and debug workflows locally on desktop in Visual Studio Code, but it was announced after I wrote the automation. I hope, it will improve overall experience building apps in Azure Portal, because this sometimes made be mad and miserable.

Decision to use Logic Apps came first after reading about Azure AD delta queries in the above mentioned B2B tutorial:

*Through the use of Delta Query, tenant admins can deploy a scripted “pull” process to automate discovery and provisioning of identities to support resource access. This process checks the home tenant for new users and uses the B2B APIs to provision those users as invited users in the resource tenant.*

Then this  brought me to another excellent blog series about using [Delta Query for Microsoft Flow](https://www.ableblue.com/blog/archive/2019/01/17/graph-delta-query-flow-part-1). It is written is so much great details that helped me a lot to understand the sequence of using Delta Query and inherit part of the logic. So, challenge was accepted.

#### Delta Query in Microsoft Graph

*Delta query enables applications to discover newly created, updated, or deleted entities without performing a full read of the target resource with every request. Microsoft Graph applications can use delta query to efficiently synchronize changes with a local data store.*

The Delta Query feature is very well documented in the [official Microsoft documentation](https://docs.microsoft.com/en-us/graph/delta-query-overview).

Basically, it is a special type of Graph API queries that returns changes (delta) since the last run. It can help therefore speed up collecting information from Azure AD when you require only changes performed on a particular object in a tenant.
The pattern working with delta queries is the following:

1. Execute a request to get the initial state of the object.
2. The response will contain the resource you requested and a state token. The token is either:
- `NextLink` indicating that there are additional pages of data
- `DeltaLink` indicating that there is no more data and the next request should use the DeltaLink to determine changes in the resource.
3. Save the `DeltaLink` and use it next time for first step to get only changes since the last run

There are two delta functions we will use for the automation:
1. [User Delta](https://docs.microsoft.com/en-us/graph/api/user-delta?view=graph-rest-1.0)
2. [Group Delta](https://docs.microsoft.com/en-us/graph/api/group-delta?view=graph-rest-1.0)

#### Overview of B2B sync automation

![High level overview of B2B sync automation](\assets\img\2021\2021-05-06\LogicAppsHLD.png)

##### Logic Apps

**Code for all Logic Apps is available in [LogicApps GitHub repo](https://github.com/Astashin/LogicApps/tree/main/Azure%20AD%20B2B%20sync%20engine).**

There are three different apps working together:
1. [B2B Invitation worker.](https://github.com/Astashin/LogicApps/blob/main/Azure%20AD%20B2B%20sync%20engine/Invitation%20Worker.json) Reads user information from source and invites to destination tenant. Saves data for attribute syncing
2. [Attribute sync worker.](https://github.com/Astashin/LogicApps/blob/main/Azure%20AD%20B2B%20sync%20engine/Attribute%20Sync%20Worker.json) Performs synchronization of user object attributes from source to destination tenant
3. [Main workflow.](https://github.com/Astashin/LogicApps/blob/main/Azure%20AD%20B2B%20sync%20engine/Main%20Workflow.json) Triggered by scheduler and starts two sync workers above. Writes logging information to storage table



##### Storage account
Storage account hosts storage table. Tables are used to store configuration information, delta links for users and groups and logging information.

##### Key Vault
Stores service principal secrets to read information from source (remote) tenants. 

##### Managed Identity
[User-assigned](https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/how-to-manage-ua-identity-portal) managed identity to assign permissions for Logic App in central (destination) tenant.

![Azure Resources](\assets\img\2021\2021-05-06\Azure Resources.png)

##### Components in source tenants
* Azure AD groups. Only users added to dedicated sync group are invited to central tenant
* Azure AD Service Principal. App registration in source Azure AD tenant to delegate permissions for automation in remote tenants to query sync group membership and user attributes.


### High-level overview of how automation works

##### 1. **Main workflow**
- Main workflow is triggered by schedule (say, each 30 minutes).
- Main workflow pulls general settings and configuration for each tenant from storage table
- For each tenant it starts B2B Invitation worker providing configuration as input parameters. Multiple workers (each per tenant) run in parallel, main workflow waits until they finish

##### 2. **B2B Invitation worker**
- receives tenant, service principal and group object IDs as input parameters. 
- gets secrets for service principals from key using service principal (application client) id
- Checks if Delta Link for sync group from remote tenant is stored in DeltaLinkGroups
- Queries sync group membership using delta link or initial delta query:
  {%- highlight ruby -%}
GET https://graph.microsoft.com/v1.0/groups/delta?$select=members&$filter=id eq 'sync_group_id'
  {%- endhighlight -%}

![Delta query](\assets\img\2021\2021-05-06\Invitation-Worker1.png)

- Actions are taken depending on conditions:
 1. If user is added to the sync group and doesn't exist in destination tenant, invitation is performed
 2. If user is added to the sync group and exists in destination tenant in Deleted Items container, account is restored
 3. If user is added to the sync group and exists in destination tenant as guest, account is converted to member
 4. If user is removed from the sync group and exists in destination tenant, external account is removed
- Each invited users are added to DeltaLinkUsers table. Entry for removed user is deleted from the table respectively

![Invitation](\assets\img\2021\2021-05-06\Invitation-Worker5.png)

- Delta query for sync group is saved to DeltaLinkGroups
- For each user object removed to sync group it deletes user to destination tenant
- Worker returns logging information for main workflow as output

##### 3. **Main workflow**
- For each tenant enabled for attribute sync starts attribute sync worker

##### 4. **Attribute sync worker**
- Reads all users from DeltaLinkUsers table. 
- If delta link is stored in the table it is used to query user object's attributes. If delta link doesn't exist in the table (first run), initial Graph API query is performed:
  {%- highlight ruby -%}
GET https://graph.microsoft.com/v1.0/users?$select=synced_attributes&$filter=id eq 'user_id'
  {%- endhighlight -%}

![Attribute Sync](\assets\img\2021\2021-05-06\attribute-sync3.png)

- Attributes returned by query are written for object in destination tenant. If there were no changes, delta query returns empty output
- If there are more than two proxyAddresses, temporarily set mail in destination tenant to be proxy address from remote tenant 
- Get direct reports for each user in remote tenant using GraphAPI query:
  {%- highlight ruby -%}
GET https://graph.microsoft.com/v1.0/users/user_id/directReports?$select=id
  {%- endhighlight -%}
- Sets manager attribute in destination tenant using GraphAPI query:
PUT https://graph.microsoft.com/v1.0/usersuser_id/manager/$ref
- Workflow returns logging information for main workflow as output

##### 5. **Main workflow**
- Saves logs from both workers to Logs table

#### Some comments and highlights of current automation

1. Sync engine works for users added to specific group in source tenants. This gives control to remote tenant's admins to control which accounts are enabled for synchronization. It is not difficult to change the automation to enumerate users using delta queries instead of groups.
2. Logic Apps does not have something like function in a programming language. Logic Apps can launch another app instead. This way repetitive actions can be optimized by parallelizing work and big workflow split into shorter parts. For this reason, sync engine is split to three parts, main one and two workers. Each worker invites or sync attributes for separate source tenant. Main workflow waits until slowest worker finishes its job.
3. Attribute sync worker performs several API queries per each user. In real time scenario it took approx. 10-15 minutes for initial sync (base attributes, proxy addresses, manager) of 500-700 users. Even without attribute changes it takes near two minutes for the same number of users. More critical here are costs of running this app. In production attribute sync worker spends for ~2000 users around 24$ per day running every 30 minutes. Indeed, code optimization might reduce these costs. Reducing frequency (once per hour) would proportionally reduce this cost.
4. In our use case, it was required to synchronize this attribute too, for most other scenarios it is probably not required. Problem is, Graph API does not support writing proxyAddress attributes. Trick is, Graph API allows to write mail attribute (although standard AzureAD PS module does not allow this). Azure AD automatically adds proxy address for each address in mail attribute. So, as a workaround, attribute sync automation temporarily set mail attribute in destination tenant for each for proxy address attribute of synced user object in source tenant. As a last step primary SMTP address is specified as mail attribute. Here problem is, proxyAddress then not removed if mail is changed. That is, synced user object can potentially have more proxy addresses than in source tenant.
5.	If full invitation sync is required for a tenant, respective entry should be removed manually from DeltaLinkGroups table. This is similar to initiating full sync in AD Connect or AD Connect cloud sync. Full sync also causes all delta links in DeltaLinkUsers table to be rewritten. Changing list of synced attributes requires removing all delta links from DeltaLinkUsers tables for a tenant, so performing manual sync would be a solution.
6.	If user object is assigned privileged role (directly or via membership in role enabled group) modifying attributes in destination tenant will fail as managed identity does not have enough permissions for this (only global administrator or similar role can modify attributes of other admins). For this reason, particular user objects could be excluded from attribute sync in table.
7. By default, loops in Logic Apps run in parallel. Using variables inside loops can cause unpredictable results to be stored in these variables. This behavior is described [in Microsoft documentation](https://docs.microsoft.com/en-us/azure/logic-apps/logic-apps-control-flow-loops#foreach-loop-sequential). This might be unexpected before you face this first time. As it's often required to store some intermediate data somewhere, using Data Compose action can be used as a workaround. This trick is used for both invitation and attribute sync workflows because running each cycle sequentially would dramatically slow down the sync engine performance.
8. Delta queries does not immediately return data for newly created Azure AD objects. From my experience, it takes around 30 seconds after creation. Usually, it is not a problem for automation. However, there are two issues detected when delta links:

-	After this [recorded incident](https://status.azure.com/en-us/status/history) in March 2021 some delta links returned empty results even when new objects were added to sync scope: **RCA - Authentication errors across multiple Microsoft services (Tracking ID LN01-P8Z)** 
-	We also experienced sporadic issue with delta queries: *"The requested media type is not supported. The allowed values are json with minimalmetadata"*. Luckily, this problem occurred very rarely.

In the next chapter I'll describe configuration of each component, describe workflows' logic in more details, describe prerequisites and installation instruction. 