---
layout: post
title:  "Single Azure AD tenant for large enterprises, part 7: Role delegation with Administrative Units"
date:   2021-05-20
description: In this part we will talk about permission delegation challenges in single Azure AD tenant and how to solve them with administrative units
categories:
  - Azure AD
tags:
  - Azure AD
  - Logic Apps
  - Administrative Unit
---

<p class="intro"><span class="dropcap">I</span>n this part we will talk about permission delegation challenges in single Azure AD tenant and how to solve them with administrative units</p>

### Role delegation in Azure AD

Often when we are dealing with single Azure AD tenant models for, crucial is how to delegate permissions to IT staff. It is important for medium size companies having first line of support and admins who wants to delegate as much as possible routine tasks to Helpdesk. In enterprises it could be several companies each having its own Helpdesk that should only be granted permissions on user objects in a tenant only within a company.

However, unlike Active Directory, Azure AD only offered flat administration model for a long time. This involved using built-in and custom global roles at global level. That is, for example, built-in User Administrator role has almost unrestricted rights over all user objects inside tenant (unless they have other privileged roles assigned). For many Active Directory administrators including me who used to hierarchy built on Organizational Units and nested security groups, that was surprise to find out that Azure AD does not have anything like that. I can only guess what the reason for Microsoft was to offer such an architecture. Maybe to reduce complexity involved with understanding exact permissions on objects when deep nesting is in place and by restricting this ability to simplify administration. Nevertheless, many customers soon realized that such design is only suitable for small tenants and requested improvements in this area. 

### Administrative units (AU) and role-assignable groups

To address lack of granular delegation, updated Administrative Units feature was introduced in 2020. I'm telling updated since admin units as a feature was available with very limited functionality without GUI support since several years already before. 
An administrative unit is an Azure AD resource that can be a container for other Azure AD resources. As of now, administrative unit can contain only users and groups. List of assignable roles is [very short](https://docs.microsoft.com/en-us/azure/active-directory/roles/admin-units-assign-roles) but should cover most commonly used delegation tasks within Azure AD, including resetting Azure MFA for users.

Another very useful feature released in 2020 was new [Azure AD group-based role assignment feature](https://docs.microsoft.com/en-us/azure/active-directory/roles/groups-concept). It allows to create role-assignable cloud-only groups and was indeed [very long-awaited feature](https://feedback.azure.com/forums/169401-azure-active-directory/suggestions/12938997-azuread-role-delegation-to-groups). 

![Role groups](\assets\img\2021\2021-05-20\rolegroups.png)

Combined with administrative units it allows to build more advanced role delegation model.

![Administrative units](\assets\img\2021\2021-05-20\AUHighlevelDesign.png)

Additionally, Privileged Identity Management can work with both features too. [Eligible roles can be activated by user accounts in role groups to and scoped to admin units.](https://docs.microsoft.com/en-us/azure/active-directory/privileged-identity-management/pim-how-to-add-role-to-user?tabs=new#assign-a-role-with-restricted-scope)

### What is still missing

Main limitation of admin units for me is lack of compatibility with M365 services (or probably that is limitation of M365 itself). There are more minor limitations such as lack of nesting or inability to add devices to admin units. However, this one might introduce extra administrative overhead: adding and removing administrative unit members dynamically based on attributes is [not supported](https://docs.microsoft.com/en-us/azure/active-directory/roles/administrative-units#administrative-unit-management).

![Not supported](\assets\img\2021\2021-05-20\AUManagement.png)

I am not aware of any plans to fulfill this gap by making it is possible to auto-populate objects to AU similar way it is done, for instance, for dynamic groups. While it is not natively available, I offer to use my automation that fulfills this gap. Similar to Azure AD B2B sync automation described in previous parts ([5](https://blog.astashin.com/blog/Bring-em-all-in-p5/), [6](https://blog.astashin.com/blog/Bring-em-all-in-p6/)) of the blog series, this one is also implemented as (our favorite) Logic Apps.

### Auto-adding users to admin units with Logic Apps

This automation is very similar to [B2B invitation workflow](https://blog.astashin.com/blog/Bring-em-all-in-p5/). It was written after this one and contains old pieces of code and inherited most of its logic. All components are the same except of missing Key Vault, since user-assigned managed identity will be used to perform privileged operations in Azure AD. 
The automation reads membership of dynamic groups. Rules for the dynamic groups reflect criteria of placing user objects to AUs. Automation queries group membership using Graph API and add or removes members to administrative units as a result of membership changes. Delta queries are in use again. Also, as before, the current automation is implemented as two Logic Apps hosted workflows: main reading configuration and writing logs and a worker workflow doing main job and triggered by the first one. 

![Auto-adding users to admin units with Logic Apps](\assets\img\2021\2021-05-20\AUAutomationDesign.png)

### High-level overview of how automation works

##### Sync groups

Synchronization groups are Azure AD dynamic group that are used to populate membership based on attributes. Here are some examples on rules we can use to distinguish users in groups and AU.

- Based on source Active Directory:

 {%- highlight ruby -%}
 user.onPremisesSecurityIdentifier -contains "S-1-5-21-11111111-1111111111-111111111"
  {%- endhighlight -%}

Since you [can't use onPremisesDistinguishedName attribute for dynamic group rules](https://feedback.azure.com/forums/169401-azure-active-directory/suggestions/39674431-dynamic-groups-add-onpremisesdistinguishedname-pr), workaround is to use AD domain's SID from onPremisesSecurityIdentifier attribute. 

- External users of type members

 {%- highlight ruby -%}
(user.userType -eq "Member") and (user.userPrincipalName -contains "remotedomain.com#EXT#@")
  {%- endhighlight -%}

If external users are invited as members to a tenant and need to be automatically placed to corresponding administrative unit

##### **Main workflow**

- Reads configuration settings for each pair of Group/AU tenant from `AdminUnitConfiguration` in storage account
- Triggers automation worker workflow in the loop for each row excluding entries where `Enabled=false`
- Saves output from worker workflow to Logs table

![Main](\assets\img\2021\2021-05-20\MainWorkflow.png)

##### **Main workflow**

- Queries DeltaLink from `DeltaLinkGroups` storage table
- If DeltaLink is not found for tenant, initial Graph API query is performed

  {%- highlight ruby -%}
https://graph.microsoft.com/v1.0/groups/delta?$select=members&$filter=id eq 'sync group id'
  {%- endhighlight -%}

- In both cases the list of members is collected and stored in an `UsersArray` array
- List of current members of administrative units is queried is stored in `AdminUnitUsersArray` array

  {%- highlight ruby -%}
 https://graph.microsoft.com/beta/administrativeUnits/AdminUnitId/members?$select=id
  {%- endhighlight -%}


- Each entry from UsersArray is compared with each member from `AdminUnitUsersArray`:
1.	Each object from `UsersArray` that doesn't exist in `AdminUnitUsersArray` is added to AU as member (add a member API)
2.	Each object from `UsersArray` with '@removed' field that also exists in `AdminUnitUsersArray` is removed from AU (Remove member API)

### Automation configuration

I won't describe automation in very much details as it is very similar to B2B invitation one. I will talk more about differences between them.
Worth mentioning, user objects could be added to administrative units even if the automation is running. If user object is not part of the sync group or automation failed for some reasons, this could be done manually in [Administrative Unit menu in Azure Portal](https://portal.azure.com/#blade/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/AdminUnit) or using ay other way. 

#### User managed identities

Create user managed identities. This time we need **Group.Read.All** and **AdministrativeUnit.ReadWrite.All**([reference](https://docs.microsoft.com/en-us/graph/permissions-reference#administrative-units-permissions)) application permissions. Note, granting write permissions to resource group where managed identity is located gives full permissions on administrative units in Azure AD tenant, so Azure IAM permissions should be assigned carefully.

 {%- highlight ruby -%}
$MSIDisplayName = "Azure-AD-Automation-MSI"
$MSIObjectId = (Get-AzureADServicePrincipal -SearchString $MSIDisplayName).ObjectId
$GraphAPIAppObject = Get-AzureADServicePrincipal -SearchString "Microsoft Graph" | Select-Object -first 1

$GraphAPIPermissions = $GraphAPIAppObject | select -ExpandProperty AppRoles | select @{n='Id';e={$_.Id}}, @{n='Permission';e={$_.Value}}

New-AzureADServiceAppRoleAssignment -ObjectId $MSIObjectId -PrincipalId $MSIObjectId -ResourceId $GraphAPIAppObjectId -Id $GraphAPIPermissions.Where{$_.Permission -eq 'Group.Read.All'}.id
New-AzureADServiceAppRoleAssignment -ObjectId $MSIObjectId -PrincipalId $MSIObjectId -ResourceId $GraphAPIAppObjectId -Id $GraphAPIPermissions.Where{$_.Permission -eq 'AdministrativeUnit.ReadWrite.All'}.id

{%- endhighlight -%}

#### Azure Storage Account Table


Three storage tables are required:

##### **AdminUnitConfiguration**

List of all sync groups and admin units the engine synchronizes user objects to. Most important sync engine configuration is stored here. Data is filled by administrator manually before starting the sync.

|     Property        |     Type       |     Description                                                             |
|---------------------|----------------|-----------------------------------------------------------------------------|
|     PartitionKey    |     String     |     Any value, for   example “AU”                                           |
|     RowKey          |     String     |     Object id of an   administrative unit in Azure AD                       |
|     GroupId         |     String     |     Object id of a   group in Azure AD to synchronize membership from       |
|     Enabled         |     Boolean    |     If Enabled is   false, no sync will be performed for a particular AU    |

##### **DeltaLinkGroups** 
Storage of delta links for dynamic sync groups. Same entries as for B2B sync workflow

|     Property        |     Type      |     Description                         |
|---------------------|---------------|-----------------------------------------|
|     PartitionKey    |     String    |     Source tenant   Id                  |
|     RowKey          |     String    |     Sync group Id   is source tenant    |
|     DeltaLink       |     String    |     Delta link to   Graph API query     |

The same way, a row (entry) could be manually removed to initiate full synchronization without using delta link.

##### **Logs** 
Stores logs of automation. Main workflow fills data in the table.

|     Property        |     Type      |     Description                                                                                                                |
|---------------------|---------------|--------------------------------------------------------------------------------------------------------------------------------|
|     PartitionKey    |     String    |     Source tenant   Id                                                                                                         |
|     RowKey          |     String    |     Random GUID                                                                                                                |
|     Action          |     String    |     Action   performed by invitation or attribute sync workflows                                                               |
|     DateTime        |     String    |     Date and time   in UTC of an action. Timestamp property is built-in property and shows when   data was written to table    |
|     ErrorMessage    |     String    |     Error or   informational message                                                                                           |
|     ObjectId        |     String    |     User Object   Id in destination tenant                                                                                     |
|     GroupId         |     String    |     Sync group objectId                                                                                                        |

![Storage](\assets\img\2021\2021-05-20\StorageTables.png)

**Code for all Logic Apps is available in [LogicApps GitHub repo](https://github.com/Astashin/LogicApps/tree/main/Azure%20AD%20Admin%20Units%20Automation).** Don't forget to change paths, connection for tables and assign managed identity to worker workflow as described in previous part.


