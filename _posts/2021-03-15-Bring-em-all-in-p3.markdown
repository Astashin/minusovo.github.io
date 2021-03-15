---
layout: post
title:  "Bring ‘em all in: designing single Azure AD tenant for large enterprises. Part 3"
date:   2021-03-06
description: In this part of our series we will talk about some considerations before enabling AAD Connect cloud sync for AD
categories:
  - Azure AD
tags:
  - Azure AD
  - AAD Connect Cloud Sync
  - Low-level
---

<p class="intro"><span class="dropcap">T</span>his part will be again mainly high-level and is about things to consider before implementing object sync from AD using AAD Connect cloud sync.</p>

#### Responsibility split between Azure AD and AD management

In classical hybrid environment, where AD Connect syncs identities, AD domain or forest and synchronization is usually managed by the same team. When you are in an environment with multiple disconnected forests, central Azure AD tenant is probably managed by a team, who has no access nor understanding about all AD environments within an enterprise. Azure AD and AD management teams are probably not aware of each other. Therefore, establishing trusted contact between these teams is probably first and very crucial step. Why is this important: Azure AD tenant administrator has control over synchronization engine and can configure pulling all objects from remote AD including password hashes for all user objects. Indeed, the Azure AD admin will not have access to these hashes and will not be able to reset passwords for synced users. Still, controlling sync agent mean getting some control over remote AD. So, again, without cooperation between teams (strengthened by corresponding monitoring) it will be difficult to manage synchronization.

#### Overall state of remote AD domains

AD forests could be promoted many years ago and inherited since then a lot of hidden problems, especially related to security. Lack of security best practices and proper identity management, poor planning of object placement and OU structure, inconsistent or missing object attributes – problems inherent almost to every AD. Especially to those small companies, where 1st level support and AD administration is performed by 1-2 people. Let be frank: more than 90% of all AD environments have severe issues and lack of best practices but identities from such forests you probably must bring to the cloud. One golden rule here: never assign highly privileged roles for hybrid accounts for identities synced from such AD forests. Basically, you should consider on-premises AD compromised and think in advance about potential lateral movement to your Azure AD tenant from attacker, having full control over remote AD forest. This and other recommendations described in detail in [Protecting Microsoft 365 from on-premises attacks blog post](https://techcommunity.microsoft.com/t5/azure-active-directory-identity/protecting-microsoft-365-from-on-premises-attacks/ba-p/1751754).
Additionally, when you start syncing from AD, you do not want to bring its chaotic state to the cloud. Even if an AD forest is in perfect shape, naming convention in multiple AD domains is most probably very different. Purpose of syncing to central tenant is usually central place collaboration, so all users within it should have seamless experience communicating with employees from one or another company. If we are talking only about users, that’s attributes like display name having the same format for everyone. Luckily, attribute mapping could help you to overcome some of the issues.

![Attribute mapping](\assets\img\2021\2021-03-15\AttributeMapping.png)

#### Non-routable domains in UserPrincipalName attribute of AD user objects

This is quite complex topic to discuss it in more details. This problem is not specific for AADC cloud sync and is well-known prerequisite before enabling every Azure AD hybrid design. Problem is, in many cases UPN of user objects in AD contain domain name of a domain. In many cases it is chosen to not be equal to company’s domain name, available in Internet. That is, company.com could be domain for company’s web site and email addresses, but AD forest could have names company.local, company.dom, and other variants. Sometimes this is done to avoid splitting DNS.
[In this article](https://docs.microsoft.com/en-us/microsoft-365/enterprise/prepare-a-non-routable-domain-for-directory-synchronization?view=o365-worldwide) Microsoft simply suggests to change UPN of all users in AD to contain domain registered in Azure AD (and therefore routable in Internet). This is probably could be done for very small number of users, but in most of the cases you cannot just change UPN attribute. It acts as primary username for AD and can potentially affect some other on-premises services, which AD admins are sometimes not aware about. Changing UPN would require extensive testing and planning, so could slow you down bringing identities to Azure AD.
Another Microsoft's recommendation is to match UPN with primary SMTP address (mail attribute in AD). This best practice is probably to make user experience easier but is not a [strict technical requirement](https://docs.microsoft.com/en-us/windows-server/identity/ad-fs/operations/configuring-alternate-login-id#applications-and-user-experience-after-the-additional-configuration). 

Yes, ideally, we would like to have UPN in AD, email address and UPN in Azure AD to be equal. But is it a strict requirement you must comply with? Answer is, not always.

Most important is matching of `UserPrincipalName` attributes. It is required when:
* Using [Seamless Single Sign-On (SSSO)](https://docs.microsoft.com/en-us/azure/active-directory/hybrid/how-to-connect-sso). SSSO is basically stretching Kerberos authentication from AD to Azure AD. Usernames in Kerberos tickets issues by domain controllers sent to AAD must contain same value as UPN for user objects. Please bear in mind, SSSO is also known as “bring all weakness of Kerberos to the cloud” since it is [prone to pass-the-hash type of attacks](https://www.dsinternals.com/en/impersonating-office-365-users-mimikatz/) that can eliminate all benefits it gives
* Environments where authentication is performed in ADFS usually use UPN attribute as identity attribute within SAML assertions. Azure AD trusts ADFS and its tokens, extract username attribute from tokens and authenticate user with the same value in UPN attribute. But as [it’s possible for ADFS to define another attribute to act as authenticator](https://docs.microsoft.com/en-us/windows-server/identity/ad-fs/operations/configuring-alternate-login-id#manually-configure-alternate-id), I am sure, it is also possible to change claim rule to include more advanced transformation expressions. 

However, when using Password Hash Sync (PHS) without SSO requirement, you could set up UPN in Azure AD to be almost everything that contains registered domain. Here is attribute mapping expressions could help a lot and I will talk about it later on. As our goal could be bringing user accounts to Azure AD tenants as soon as possible, we might want to define new common format of UPN attribute. Later, UPN could be changed in AD domains if required. Here one very important thing to consider: you should plan UPN format from the beginning. Changing them later could cause issues in different M365 services, [especially OneDrive](https://docs.microsoft.com/en-us/onedrive/upn-changes).

To summarize:
* It is important to think in advance about UPN attribute format in Azure AD as changing it later can cause M365 services and sign-in to federated applications to fail
* UPN in AD and Azure AD could be different if PHS is used without Seamless SSO
* We can form UPN in AAD using mapping expressions in AAD Cloud Sync and change it in AD later, if necessary

#### Examples of attribute mapping for most commonly used Azure AD attributes

##### 1. DisplayName

* Form display name from first and last name
  {%- highlight ruby -%}
  Join(" ", Trim([givenName]), Trim([sn]))
  {%- endhighlight -%}

* Form display name from first and last name, and append company name
  {%- highlight ruby -%}
    Append(Join(" ", Trim([givenName]), Trim([sn])), " (Company)")
  {%- endhighlight -%}

* Form display name from first and last name. If last name is missing, add “Mailbox” to account’s display name. This might be handy if you sync shared mailboxes with names like “Invoice”, “Info” or such and want to distinguish them from user accounts

  {%- highlight ruby -%}
  IIF(ToLower([extensionAttribute1], )="sharedmailbox", "false", IIF(IsPresent([userAccountControl]), IIF(BitAnd([userAccountControl], 2)="0", "True", "False"), Not([accountDisabled])))
  {%- endhighlight -%}

##### 2. AccountEnabled

* Sometimes you need to synchronize objects that normally should not sign-in to Azure AD. If such accounts have "sharedmailbox" in extensionAttribute1, you can use the following expression to disable them in Azure AD

  {%- highlight ruby -%}
  IIF(ToLower([extensionAttribute1], )="sharedmailbox", "false", IIF(IsPresent([userAccountControl]), IIF(BitAnd([userAccountControl], 2)="0", "True", "False"), Not([accountDisabled])))
  {%- endhighlight -%}

##### 3. UserPrincipalName

We will describe later in more details why configuring attribute mapping expressions for UserPrincipalName (UPN) might be very important. Below are examples how to form UPN in AAD that should have `firstname.lastname@domain.com` format

* Form UPN in Azure AD as prefix of UPN in AD and domain registered in Azure AD

  {%- highlight ruby -%}
  Append(Item(Split([userPrincipalName], "@"), 1), "@domain.com")
  {%- endhighlight -%}

* Form UPN when mail address for object has name.lastname and SamAccountName has firstname_lastname format

  {%- highlight ruby -%}
IIF(IsPresent([mail]), Append(Item(Split([mail], "@"), 1), "@domain.com"), Append(Replace([sAMAccountName], "_", , , ".", , ), "@domain.com"))
  {%- endhighlight -%}

* If ExtensionsAttribute1 stores information if a user object is in fact shared mailbox, add company-mbx to sAMAccountName to avoid duplicate UPN attribute conflicts. If it is not shared mailbox, form UPN with first and last name attributes

  {%- highlight ruby -%}
IIF([ExtensionAttribute1]="SharedMailbox", Append([sAMAccountName], ".company-mbx@domain.com"),  Append(ToLower(NormalizeDiacritics(StripSpaces(Join(".", [givenName], [sn])))), "@domain.com"))
  {%- endhighlight -%}


#### Some tips for editing attribute mapping expressions 

As you can see, mapping expressions could be quite complex. To debug them before using in productions, especially if it comes to for critical attributes like UPN, you can use two ways:
1.	Use User Provision on Demand feature to sync only single object and this way apply attribute mapping only to it. ![Provisioning on demand](\assets\img\2021\2021-03-15\AttributeMapping2.png)However, sometimes it is required to test expression for large amount of synced objects

2.	Then you can first apply mapping for non-critical attributes, for such as ExtensionAttributeN. Then query it from Azure AD for all affected users and check if all values are expected. However, there is a small problem using `ExtensionAttribute`: standard AAD PowerShell modules do not allow to query these attributes. As a workaround you might need to write a script querying Graph API directly

