---
layout: post
title:  "Single Azure AD tenant for large enterprises, part 3: Azure AD Viral tenants"
date:   2021-05-04
description: In this part of our series we will talk about viral tenants and external users
categories:
  - Azure AD
tags:
  - Azure AD
  - AAD Connect Cloud Sync
  - Viral Tenant
---


<p class="intro"><span class="dropcap">We</span>have the same goal: to bring disconnected AD domains to a central tenant. But imagine a scenario: you contacted local AD team and they assured that they don't have Azure AD instance in place. You then try to register and verify DNS domain in the central tenant. Surprisingly, but this operation fails as the domain is registered in some other tenant. Quick search in Internet gives the answer: we are dealing shadow (or viral) tenant.</p>

#### Troubles with hidden viral tenants along the way to central one

This concept is mentioned in several articles: [1](http://1.https//docs.microsoft.com/en-us/azure/active-directory/enterprise-users/directory-self-service-signup), [2](https://docs.microsoft.com/en-us/azure/active-directory/enterprise-users/domains-admin-takeover), [3](https://docs.microsoft.com/en-us/azure/active-directory/external-identities/user-properties), [4](https://docs.microsoft.com/en-us/azure/active-directory/external-identities/faq).

However, I think, most of AAD administrators first learn about shadow tenants after facing the aforementioned issue. Basically, unmanaged tenants are created every time during Self-service sign-up for a cloud service if email's domain is registered in an Azure AD tenant. Probably to address confusion and problems associated with viral tenants, Microsoft decided to stop creating new viral tenants starting **October 2021** (initial plan was **March 31**).

Easy way to determine if a domain is registered in a viral tenant is to open URL in the following format:

  {%- highlight ruby -%}
https://login.microsoftonline.com/common/userrealm/name@domaintoverify.com?api-version=2.1
  {%- endhighlight -%}

If returned JSON contains `"IsViral":true`, domain is registered in viral tenant.

But what's the problem with shadow tenants? If tenant exists, there are users registered to Microsoft services. (such as Teams), but it's problematic to find out where which exactly. It means, users might have files and conversation they could lose if tenant is removed. Therefore obvious decision would be to perform external takeover as described here, as this method preserves user's accesses and data. External takeover moves users from unmanaged to central one remaining data in cloud services intact (not all services are supported for the takeover).
After takeover is performed as described accounts will be moved to destination tenant after some time. Based on my experience it happens in around 10–15 minutes. After move users accounts appear in destination tenant with [Microsoft Account as a source](https://docs.microsoft.com/en-us/azure/active-directory/external-identities/user-properties#source).
However, there is undocumented behavior of this move process you need to be aware of:

##### If a moved user account has UPN or mail address (these attributes are always equal in unmanaged tenant) overlapping with user account in destination tenant this account would not be moved

Moreover, you would lose this account and all associated data (I don't know if there is way to get this data through vendor's support). Therefore, my tip is to always start synchronization from AD only after a viral domain has been taken over to a destination tenant. Also, before the takeover try to sign-in to shadow tenant using account with email address registered there. Although tenant is unmanaged it's possible to open Azure Portal and see all user accounts. Also note, if you perform internal takeover and become global admin role in shadow tenant, no external takeover would be possible in the future (as a tenant is not considered unmanaged anymore).


#### Merging external accounts

Azure AD user account merging (or matching) is relatively known process although, again, not very well documented by Microsoft itself. Matching is a process of instructing AD Connect or AD Connect cloud sync to sync user account from AD to existing account in Azure AD. In case of hard-matching `immutableID` attribute in Azure AD is filled with Base64 encoded value of Active Directory user's `objectGUID` attribute:

  {%- highlight ruby -%}
set-azureaduser -usertype Member -immutabelID "immutabelID"
  {%- endhighlight -%}

Here is one of the best articles describing the matching [process in details](https://dirteam.com/sander/2020/03/27/explained-user-hard-matching-and-soft-matching-in-azure-ad-connect/).

But what if existing external user is merged with AD synced user account? Then it has two sources in Azure AD:

![Multiple sources](\assets\img\2021\2021-05-04\1.png)

1. External account from another Azure AD tenant ![External account](\assets\img\2021\2021-05-04\2.png)

2. Merging with Microsoft Account user type (could be performed after viral tenant takeover) ![Microsoft account](\assets\img\2021\2021-05-04\3.png)

3. Merging with guest account invited [using one-time passcode](https://docs.microsoft.com/en-us/azure/active-directory/external-identities/one-time-passcode)

![OTP account](\assets\img\2021\2021-05-04\4.png)

You can merge external account with one from AD using hard matching, that is filling ImmutableId attribute. If soft matching is performed by AD Connect for external accounts too? That's the good question and I observed how OTP invited guests suddenly became having two sources of authority (case 3) without any modifications from my end.
Although technically it's possible to merge external and AD accounts in AAD, I won't recommend doing this wherever possible. First and foremost, as it's undocumented feature, you could come across some unexpected behavior. Here are some issues I come across hard matching external accounts:

* Sign-in to Azure Active Directory using email as an alternate login ID may not properly work
* Using email as identifier for SAML federation it might come to authentication failures

![SAML claims](\assets\img\2021\2021-05-04\5.png)

* Error "E-Mail OTP user cannot sign in with local password" (code 500346) when OT user is was merged with AD account and tries to sign-in using UPN and password.

Last one basically prevents users to sign-in to Azure AD using standard way. This error started to occur in February 2021 after Microsoft configured disabled ability to sign-in using both methods (OTP and password) for such guest accounts:

*Previously, it was possible for those users to sign in using either credential type, but the change we're enforcing is to only allow users configured for email one-time passcode authentication to sign in using email one-time passcode. This change will go into effect starting Monday February 8th in some regions and in all regions by Monday February 15th, 2021.*

#### Scripts

In the end of this post let me share some scripts you might find useful.

##### 1. PowerShell function to match user account in AAD using objectGUID in AD.

  {%- highlight ruby -%}
function set-immutableIdFromAzureADUsingGUID ($UserPrincipalName, $GUID)
{
Get-AzureADUser -Filter "userprincipalname eq '$UserPrincipalName'" | %{Set-AzureADUser -ObjectId $_.ObjectId -ImmutableId ([system.convert]::ToBase64String([Guid]::Parse($GUID).tobytearray())) -UserType member}
}

  {%- endhighlight -%}

Usage:

  {%- highlight ruby -%}
set-immutableIdFromAzureADUsingGUID -UserPrincipalName name@tenant.cloud' -GUID 'ObjectGUID'
  {%- endhighlight -%}


##### 2. PowerShell function to match user account in AAD with AD synced one. Scenario is: merge accounts after synchronization from AD has been enabled. **Warning**: script removes synced account to finish matching

  {%- highlight ruby -%}
function switch-immutableIdFromAzureAD ($CloudUserPrincipalName, $HybridUserPrincipalName)
{
try{
$OldUserObject = Get-AzureADUser -Filter "userprincipalname eq '$HybridUserPrincipalName'"
}
catch{
Write-Host "Hybrid account $HybridUserPrincipalName was not found in AAD"
}
try{
if ((Get-AzureADUser -Filter "UserPrincipalName eq '$CloudUserPrincipalName'").count -ne 0 -and $OldUserObject.cound -ne 0){
Remove-AzureADUser -ObjectId ($OldUserObject.ObjectId)
Start-Sleep -s 30
Remove-AzureADMSDeletedDirectoryObject -Id ($OldUserObject.ObjectId)
Start-Sleep -s 10
Set-AzureADUser -ObjectId $CloudUserPrincipalName -ImmutableId ($OldUserObject.ImmutableId) -UserType member
Write-Host "Cloud account $CloudUserPrincipalName has been merged with $HybridUserPrincipalName"
}
}
catch{
Write-Host "Account $CloudUserPrincipalName have not been found in Azure AD or other problem during merging"
}
}
  {%- endhighlight -%}

Usage: merge account with UPN `name@tenant.com` with existing cloud only account `name@tenant.cloud`

  {%- highlight ruby -%}
switch-immutableIdFromAzureAD -CloudUserPrincipalName 'name@tenant.cloud' -HybridUserPrincipalName 'name@tenant.com'
  {%- endhighlight -%}

##### 3. Search for AAD user account having multiple sources of authority

    {%- highlight ruby -%}
Get-AzureADUser -All $true | where{$_.DirSyncEnabled -eq $true -and $_.CreationType -eq 'Invitation'} | select mail,UserPrincipalName
  {%- endhighlight -%}