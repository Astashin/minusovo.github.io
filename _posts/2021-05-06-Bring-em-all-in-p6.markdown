---
layout: post
title:  "Single Azure AD tenant for large enterprises, part 6: B2B sync engine in details"
date:   2021-05-06
description: In this part I will explain configuration of Logic Apps based B2B sync engine described in the previous part in more details
categories:
  - Azure AD
tags:
  - Azure AD
  - Logic Apps
  - Azure AD B2B
---

<p class="intro"><span class="dropcap">I</span>n this part I will explain configuration of Logic Apps based B2B sync engine described in the previous part in more details</p>

### Authentication

Sync engine contains several Azure resources and uses Graph API to communicate with Azure AD. Depending on type and location (local or remote tenant) different authentication types are used.

##### 1. **User Managed Identity**
- Graph API for local (destination) tenant. 

![Graph API for local tenant](\assets\img\2021\2021-05-12\auth1.png)

OAuth2 token audience for Graph API is **https://graph.microsoft.com**

- Key Vault

![Key Vault](\assets\img\2021\2021-05-12\auth2.png)

OAuth2 token audience for Key Vault REST API is **https://vault.azure.net**

##### 2. **OAuth2 with service principal's secret**
Azure AD authentication with service principal is used to authenticate to remote tenant with OAuth2

![OAuth2 with service principal's secret](\assets\img\2021\2021-05-12\auth3.png)

##### 3. API connection (shared key)

For simplicity queries to Azure Storage tables are performed not via REST API but using built-in connector. This connector doesn’t allow to use managed identity for authentication.

![API connection ](\assets\img\2021\2021-05-12\auth4.png)

### Preparing synchronization components in Azure

You need Azure subscription linked to an Azure AD tenant that is a destination for synced accounts. Create all resources in a dedicated resource group.

#### User managed identities

Create user assigned managed identity [in Azure Portal](https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/how-to-manage-ua-identity-portal) or any other method. Then assign application [User.ReadWrite.All](https://docs.microsoft.com/en-us/graph/permissions-reference#user-permissions) Graph API permission. Granting permissions for managed identity (which is Azure managed service principal) is not available in the  Portal but could be done using PowerShell script ([reference](https://docs.microsoft.com/en-us/azure/app-service/scenario-secure-app-access-microsoft-graph-as-app?tabs=azure-powershell%2Ccommand-line#grant-access-to-microsoft-graph)):

 {%- highlight ruby -%}
$MSIDisplayName = "Azure-AD-Automation-MSI"
$MSIObjectId = (Get-AzureADServicePrincipal -SearchString $MSIDisplayName).ObjectId
$GraphAPIAppObject = Get-AzureADServicePrincipal -SearchString "Microsoft Graph" | Select-Object -first 1

$GraphAPIPermissions = $GraphAPIAppObject | select -ExpandProperty AppRoles | select @{n='Id';e={$_.Id}}, @{n='Permission';e={$_.Value}}

New-AzureADServiceAppRoleAssignment -ObjectId $MSIObjectId -PrincipalId $MSIObjectId -ResourceId $GraphAPIAppObjectId -Id $GraphAPIPermissions.Where{$_.Permission -eq 'Group.Read.All'}.id
New-AzureADServiceAppRoleAssignment -ObjectId $MSIObjectId -PrincipalId $MSIObjectId -ResourceId $GraphAPIAppObjectId -Id $GraphAPIPermissions.Where{$_.Permission -eq 'User.ReadWrite.All'}.id

{%- endhighlight -%}

![user assigned managed identity](\assets\img\2021\2021-05-12\auth5.png)

#### Key Vault

Create Key Vault using favorite method. Additional configuration: 
- Change permission model to Azure role-based access control (preview)
- Assign Key Vault Secrets User (preview) role for user-managed identity we created 

![Key Vault](\assets\img\2021\2021-05-12\auth5.png)

#### Azure Storage Account Table

Create Azure Storage in resource group using method of your choice. Easiest way is again in Azure Portal.

To edit rows and manage tables in Azure storage accounts, the free [Azure Storage Explorer](https://azure.microsoft.com/en-us/features/storage-explorer/tool) could be used. After launching the tool, sign-in to Azure AD account with appropriate permissions.

Five tables need to be created manually:

##### **ApplicationConfiguration.** 
Contains general configuration for our sync engine. Data is filled ny administrator manually. 
Main purpose of this table is to avoid using variable constants in Logic Apps. It contains only one entry with two properties

|     Property                          |     Type      |     Description                                                                     |
|---------------------------------------|---------------|-------------------------------------------------------------------------------------|
|     DestinationTenantPrimaryDomain    |     String    |     Primary domain of   a destination tenant (to form UPN during attribute sync)    |
|     KeyVaultURL                       |     String    |     URL of Key Vault   storing service principal id and secrets                     |
|     PartitionKey                      |     String    |     Various value                                                                   |
|     RowKey                            |     String    |     Various value                                                                   |

##### **DeltaLinkGroups.** 
Storage of delta links for sync groups in remote tenants. Workflows fill data inside.

When synchronization is running, delta links for sync groups are stored in the DeltaLinkGroups table. The table has the following properties

|     Property        |     Type      |     Description                         |
|---------------------|---------------|-----------------------------------------|
|     PartitionKey    |     String    |     Source tenant   Id                  |
|     RowKey          |     String    |     Sync group Id   is source tenant    |
|     DeltaLink       |     String    |     Delta link to   Graph API query     |

To start initial full sync for a particular tenant, remove corresponding entity from DeltaLinkGroups. This will initiate invitation workflow to go through every member in sync group and rewrite delta links in DeltaLinkUsers.

##### **DeltaLinkUsers.** 
Storage of delta links of synced user objects in destination tenant. Workflows fill data inside.
DeltaLinkUsers has the following properties:

|     Property        |     Type      |     Description                                         |
|---------------------|---------------|---------------------------------------------------------|
|     PartitionKey    |     String    |     Source tenant   Id                                  |
|     RowKey          |     String    |     User object id   in source tenant                   |
|     DeltaLink       |     String    |     Delta link to   Graph API query                     |
|     Mail            |     String    |     Mail   attribute of user object in source tenant    |
|     UserId          |     String    |     User object id   is destination tenant              |

To manually initiate full attribute sync for a particular user, DeltaLink property can be removed from the table. **Important:** not the whole entity but only the property should be removed. Removing complete row for a user would stop attribute sync for user until invitation workflow restores the record.

##### **Logs.** 
Stores logs of automation. Workflows fill data inside.

|     Property        |     Type      |     Description                                                                                                                |
|---------------------|---------------|--------------------------------------------------------------------------------------------------------------------------------|
|     PartitionKey    |     String    |     Source tenant   Id                                                                                                         |
|     RowKey          |     String    |     Random GUID                                                                                                                |
|     Aciton          |     String    |     Action   performed by invitation or attribute sync workflows                                                               |
|     DateTime        |     String    |     Date and time   in UTC of an action. Timestamp property is built-in property and shows when   data was written to table    |
|     ErrorMessage    |     String    |     Error or   informational message                                                                                           |
|     ObjectId        |     String    |     User Object   Id in destination tenant                                                                                     |
|     TenantName      |     String    |     Name of the   tenant as appears in RemoteTenantConfiguration table                                                         |
|     UserMail        |     String    |     Email used to   invite user                                                                                                |

##### **RemoteTenantConfiguration.** 
List of all tenants the engine synchronizes user objects from. Most important sync engine configuration is stored here. Data is filled by administrator manually before starting the sync.

|     Property                  |     Type       |     Description                                                                                                                                                         |
|-------------------------------|----------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|     PartitionKey              |     String     |     Various value,   for example B2B                                                                                                                                    |
|     RowKey                    |     String     |     Remote tenant   name (to identify easier)                                                                                                                           |
|     TenantId                  |     String     |     Tenant Id of   source (remote) tenant                                                                                                                               |
|     ApplicationId             |     String     |     Application Id   of service principal in source tenant to perform Graph API calls                                                                                   |
|     GroupId                   |     String     |     Group Id of   synchronization group                                                                                                                                 |
|     EmailDomain               |     String     |     List of all allowed   domains used for primary SMTP addresses of users in source tenant. If there   is more than one, they should be separated using ‘,’ or ‘;’     |
|     InvitedUserType           |     String     |     Must be member   or guest. This is InvitedUserType property of Invitation Graph API type                                                                            |
|     SendInvitationMessage     |     Boolean    |     Indicates   whether an email should be sent to an invited user or not. This is SendInvitationMessage   property of Invitation Graph API type                        |
|     InviteRedirectUrl         |     String     |     The URL the   user should be redirected to once the invitation is redeemed. This is InviteRedirectUrl   property of Invitation Graph API type                       |
|     CustomizedMessageBody     |     String     |     Customized   message body. This is customizedMessageBody property of invitedUserMessageInfo Graph API   object                                                      |
|     DisplayName               |     String     |     Display name suffix   of invited user (obsolete)                                                                                                                    |
|     Enabled                   |     Boolean    |     If Enabled is   false, no sync will be performed                                                                                                                    |
|     AttributeSyncEnabled      |     Boolean    |     If Enabled is   false, no attribute sync will be performed (only invitation)                                                                                        |
|     SyncedAttributes          |     String     |     Comma separated   list of synced attributes                                                                                                                         |
|     AttributeSyncExclusion    |     String     |     Comma separated   list of object Ids excluded from attribute sync                                                                                                   |

![Storage account](\assets\img\2021\2021-05-12\storage.png)

Properties of Invitation Graph API type are described in [this Microsoft article](https://docs.microsoft.com/en-us/graph/api/resources/invitation?view=graph-rest-1.0).

Here is an example of most common attributes that could be specified in `SyncedAttributes` property:

 {%- highlight JSON -%}
 City,CompanyName,Country,Department,DisplayName,GivenName,employeeId,JobTitle,Mail,mobilePhone,officeLocation,OtherMails,PostalCode,PreferredLanguage,ProxyAddresses,State,StreetAddress,Surname,businessPhones
{%- endhighlight -%}

As changing attributes of user with Azure AD roles is a sensitive operation, user.readwrite.all permission will not be sufficient for managed identity. Therefore, primary purpose of AttributeSyncExclusion is to exclude external users who have privileged roles in destination tenant and to avoid failing attempts to sync attributes.



### Configuring remote Azure AD tenants for sync
In source tenants only two objects should be created in Azure AD.

1.	**Sync group.** Members of the group will be invited to destination tenant. This is a security group in Azure AD. Object id of this group should be stored in RemoteTenantConfiguration table.

2.	**Service principal.** Easiest way is to create it in Portal [as described here](https://docs.microsoft.com/en-us/azure/active-directory/develop/quickstart-register-app#register-an-application). Add client secret as credentials, copy it along with service principal application id and tenant id. 

Grant these application (not delegated) permissions for the application [as described here](https://docs.microsoft.com/en-us/azure/active-directory/develop/quickstart-configure-app-access-web-apis#application-permission-to-microsoft-graph) and grant admin consent:

**Microsoft Graph Group.Read.All**
**Microsoft Graph User.Read.All**

 ![Key Vault](\assets\img\2021\2021-05-12\servicePrincipal.png)

Service principals’ secrets are placed into Key Vault in destination tenant with a name equal to application Id (not a tenant id!) of corresponding service principal object.
 
 ![Key Vault](\assets\img\2021\2021-05-12\KeyVault.png)

### Final step: preparing Logic Apps.

Create API connection to storage account created before. It’s easy to perform by choosing any actions from Azure Table Storage connector and then clicking on Add new connection.

![MSI](\assets\img\2021\2021-05-12\APIConnection.png)

Create three empty Logic Apps. Configure each of them to use previously created user managed identity.

![MSI](\assets\img\2021\2021-05-12\LogicAppsMSI.png)

For additional security both workers apps can be configured for being triggered only from other LogicApps

![MSI](\assets\img\2021\2021-05-12\LogicAppsControl.png)

Open code view of both workers and paste code from GitHub. At the very bottom change parameters for `$connections` to contain proper path to Azure Table connector. For main workflow change also workflow IDs (resource Id that is full path to Logic Apps workers).

### Sync Engine Monitoring

Logs table gives comprehensive information about how invitation and sync attributes workflows worked.

![Logs table](\assets\img\2021\2021-05-12\Logs.png)

Azure AD audit log is way to monitor all cha at tenant level that includes also activities from sync engine. Note, managed identity is shown as subject that performed changes

![Azure AD Audit](\assets\img\2021\2021-05-12\AzureLogs.png)
