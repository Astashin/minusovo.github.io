---
layout: post
title:  "Bring ‘em all in: designing single Azure AD tenant for large enterprises. Part 2"
date:   2021-03-06
description: In this part of our series we will discover sync job schema and explain why we would need to change it
categories:
  - Azure AD
tags:
  - Azure AD
  - AAD Connect Cloud Sync
  - Low-level
---

<p class="intro"><span class="dropcap">S</span>hortly after release AAD Connect Cloud Sync did not allow to perform one very important thing: modifying attribute mappings in Azure Portal.</p>

The only supported way to change attribute mapping is direct modification of sync job schema, as it was described at that time in the official documentation:
*The cloud provisioning configuration creates a service principal. The service principal is visible in the Azure portal. You should not modify the attribute mappings using the service principal experience in the Azure portal. This is not supported.*

#### "Is it officially supported"?
Before we continue talking about Cloud sync job schema, I would like to give my opinion on frequently asked question about supportability. This is very important, especially if you are dealing with very new service or product. What does it mean a particular configuration is “unsupported”?  From my experience working as support engineer at Microsoft, there is in single understanding about it even inside the corporation. Being myself a Premier support engineer, I was loved to find out that customer performed something Microsoft defined as unsupported. This allows simply close a support ticket and doesn’t deal with complex situation. So, if you are part of IT administration team, this term could sound scary. Do I lose all support immediately if perform something that is not clearly described in one of numerous Microsoft articles? Well, not necessarily. 
There are two kinds of “unsupported”:
1.	**Configuration that proved to cause issues or do not work.** At the time of the writing one of examples could be using [synchronization configuration scoped to a group](https://docs.microsoft.com/en-us/azure/active-directory/cloud-sync/how-to-configure#scope-provisioning-to-specific-users-and-groups) with more than 1500 members. [Official documentation says](https://docs.microsoft.com/en-us/azure/active-directory/cloud-sync/reference-cloud-sync-faq): *when you use the group scope filtering, we recommend that you keep your group size to less than 1500 members. The reason for this is that even though you can sync a large group as part of group scoping filter, when you add members to that group by batches of greater than 1500, the delta synchronization will fail.*  The description is a bit misleading: in fact,sync profile will be put to quarantine immediately after you try to add more than 1500 users to a sync group (as of March 20201). But since the documentation at least warns you about possible impact, this is a non-working “not supported”.
2.	**Configurations that were not tested (enough) by a vendor.** Because in many cases it is not possible to test all possible scenarios a customer would like to implement, a vendor tries to stay (and keep customer) at a safe side. So, Microsoft tells “it has not been tested therefore we won’t support it”. From my perspective, this is what most of “unsupported” cases are. And this could be a huge *“grey”* area. What if a product allows to customize it in numerous ways but it is nearly impossible to document each and every scenario? Is it tested enough to be “supported” or not?  Yes, it is risky to use in production something that nobody else tested before. But if you must use product at its early stage and documentation is not complete enough. Should it really stop you from achieving your goals? 
My opinion/wish is, Microsoft and other vendors would stop using scary “unsupported” term to something that was not completely tested. I would love to hear “we didn’t test this configuration but will do our best to support customers who took a risk to use it in production” kind of statement instead. From my experience, if something is explicitly not defined as unsupported in a documentation, support services would anyway to work on issue. There is simply no authoritative source to rely on to close support request without working. But, of course, it’s always up to vendor.

#### Understanding sync job schema
The regrets I mentioned in the first part were primarily connected with this limitation. In next part of the series, I’ll explain why changing default mapping expression is so important. But before 15.10.2020 the only supported way to modify mapping was using Graph API. First you need to get two parameters from Azure Portal:

1. **Service principal ID.** In Azure Portal navigate to [Azure AD Connect menu](https://portal.azure.com/#blade/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/AzureADConnect) and choose *Manage Azure AD cloud sync* link. Choose View provisioning logs for a sync configuration you're interested in.

![](\assets\img\2021\2021-03-06\SP0.png)

Copy application Id from filter field.

![](\assets\img\2021\2021-03-06\SP.png)

2. **Provisioning Job Id.** While being in Azure AD Connect cloud sync window, click on Healthy (or whatever state your sync profile currently is) under Status column for the synchronization profile.

![](\assets\img\2021\2021-03-06\AADJobStatus.png)

You also can use a method described in [Understand the Azure AD schema article](https://docs.microsoft.com/en-us/azure/active-directory/cloud-sync/concept-attributes#view-the-schema) but it’s more complicated.

After you know service principal ID and job id we can query and edit sync schema using Graph API. For this:

1. Get current sync job scheme using Graph API by sending GET request (om [Graph Explorer](https://developer.microsoft.com/graph/graph-explorer), Postman, or similar tool) to  URL of the following format: 
`https://graph.microsoft.com/beta/servicePrincipals/<ServicePrincipalId>/synchronization/jobs/AD2AADProvisioning.<jobId>/schema`

2. The request will return schema in JSON format. Copy it to a text editor that simplifies editing JSON (Visual Studio Code, Notepad++, etc.)

3. Change the scheme. Copy JSON as body for HTTP PUT request using URL for synchronization job from p.1

Main problem is that a scheme contains more than 20 thousand lines of code. Understanding its format was crucial as it's unknown if you changed it correctly before saving it using PUT request.

#### Description of scheme format

Let us briefly describe schema format only mentioning most important parts we can change. Every schema starts with three keys:

- **@odata.context**  context URL for OData protocol
- **id** Sync Job Id 
- **Version** contains date and time when schema was saved last time

Then follows `synchronizationRules` array, that in turn contains `objectMappings` array. `objectMappings array` contains attribute mappings (inside `attributeMappings` array) for four source AD objects: contact, group, inetOrgPerson, user. As Cloud Sync can only synchronize from Active Directory to Azure AD, all keys related to source (`sourceDirectoryName`, `sourceObjectName`, and object inside `attributeMappings` array) represent objects and attributes in AD. The same way, keys related to destination (`targetDirectoryName`, `targetObjectName`, `targetAttributeName`) represent data in Azure AD. One important note here: as object class `inetOrgPerson` is only defined in AD, it mapped to objects of user class in Azure AD. I will explain later why it’s important to know.

![synchronizationRules array](\assets\img\2021\2021-03-06\AADCSScheme1.png)

After synchronizationRules array follows `directories@odata.context` key and `directories` array. The array contains two members "Active Directory" and "Azure Active Directory". Each directory array member contains attribute definitions for four object classes we previosuly talket about. Here, again, for Azure AD there is `inetOrgPerson` class and user class is described twice. There is also special class `SyncCredentialsChangeItem` that is probably used to synchronize password hashes.

![Directories definitions](\assets\img\2021\2021-03-06\AADCSScheme2.png)

To understand how to modfy expressions in directly in schema, let's take Relatively simple expression: **Left(Trim([description]), 448)**. This is how it's described in JSON:

{%- highlight ruby -%}
{
"source": {
    "expression": "Left(Trim([description]), 448)",
    "name": "Left",
    "type": "Function",
    "parameters": [
        {
            "key": "source",
            "value": {
                "expression": "Trim([description])",
                "name": "Trim",
                "type": "Function",
                "parameters": [
{
    "key": "source",
    "value": {
    "expression": "[description]",
    "name": "description",
    "type": "Attribute",
    "parameters": []
    }
}
                ]
            }
        },
        {
            "key": "length",
            "value": {
                "expression": "\"448\"",
                "name": "448",
                "type": "Constant",
                "parameters": []
            }
        }
    ]
}
}
{%- endhighlight -%}

That was indeed a big relief, when ability to edit attribute expressions in Azure Portal was finally added in October 2020. However, there are still reasons why you might still want to edit it through Graph API.

#### Customizing sync job schema: adding custom attributes to 

One of these reasons is using extension attribute (properties) AD Connect cloud sync. Imagine, you have custom application that requires to store some information in extended attributes of user and group objects in Azure AD. Extension attributes offer a convenient way to extend your Azure AD directory with new attributes that you can use to store attribute values for objects in your directory. We can create extension properties by [following step-by-step guide](https://docs.microsoft.com/en-us/powershell/azure/active-directory/using-extension-attributes-sample?view=azureadps-2.0#create-a-new-extension-property) for user and group objects.

You can use this PowerShell script to create new extended attributes for group and user objects:

{%- highlight powershell -%}
$MyApp = (New-AzureADApplication -DisplayName "Application extension attributes" -IdentifierUris "https://URL").ObjectId
New-AzureADServicePrincipal -AppId (Get-AzureADApplication -SearchString "Proofpoint extension attributes").AppId
$TargetObjects = New-Object System.Collections.Generic.List[string]
$TargetObjects.Add('User')
$TargetObjects.Add('Group')
New-AzureADApplicationExtensionProperty -ObjectId $MyApp -Name "Attribute1" -DataType "String" -TargetObjects $TargetObjects

{%- endhighlight -%}

Output of the script above would display name of extension attributes. Naming format for such attributes is “extension_<applicationObjectId>_<AttributeName>”. To get names for extension attributes after creation, following commandlet could be run: Get-AzureADExtensionProperty.
Then we can add these new attributes to schema of Cloud sync job and therefore sync attributes of user and group objects to these custom attributes. Extend schema by adding following section to `directories.name[“Azure Active Directory”].objects.name[“user”]` and `directories.name[“Azure Active Directory”].objects.name[“group”]`:

{%- highlight powershell -%}
"attributes": [
    {
        "anchor": false,
        "caseExact": false,
        "defaultValue": null,
        "flowNullValues": false,
        "multivalued": false,
        "mutability": "ReadWrite",
        "name": "extension_YourApplicationObjectId_AttributeName",
        "required": false,
        "type": "String",
        "apiExpressions": [],
        "metadata": [],
        "referencedObjects": []
    }
{%- endhighlight -%}

![Extended attributes in sync job schema](\assets\img\2021\2021-03-06\AADCSScheme3.png)

After saving job schema you will be able to manage mapping for the extended attribute in standard way.

![Extended attributes in sync job schema](\assets\img\2021\2021-03-06\AADCSScheme4.png)

This is actually not possible to achieve with classical AD Connect. Big difference is, you can only choose extra attributes you want to sync from Active Directory in configuration wizard and let it to register Azure AD application for extensions. Big difference is, you cannot register application by your own before and after this configure AD Connect to sync AD attributes to these extended ones in Azure AD. [Microsoft documentation](https://docs.microsoft.com/en-us/azure/active-directory/hybrid/how-to-connect-sync-feature-directory-extensions) explicitly explains it: 
*It is not supported to sync attribute values from AADConnect to extension attributes that are not created by AADConnect. Doing so may produce performance issues and unexpected results. Only extension attributes that are created as shown in the above are supported for synchronization.*