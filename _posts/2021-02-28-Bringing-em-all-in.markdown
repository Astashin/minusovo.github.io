---
layout: post
title:  "Bringing ‘em all in. Part 1: Large enterprise in single Azure AD tenant"
date:   2021-02-28
description: On last Friday of January 2020, Microsoft announced general availability Azure AD Connect cloud sync feature. For us as very early adopters of this new synchronization service this was not something very special
---

<p class="intro"><span class="dropcap">O</span>n last Friday of January 2020, Microsoft announced general availability Azure AD Connect cloud sync feature. For us as very early adopters of this new synchronization service this was not something very special: we started to use it in production in 2020 shortly after it preview release. After one year we brought more than 20 companies and ten thousand user accounts to the central Azure AD tenant. That was expectedly challenging journey as it usually happens when adopting a product in its early stages. In this blog series I describe all possible problems and solutions, share my thoughts experience and automation code that hopefully can help other to implement the similar project too.</p>

#### Intro 
Imagine, you are mid to enterprise level company in 2019 who heavily on Microsoft products on its on-premises systems. You are not just one monolithic firm but a holding containing dozens of other smaller companies. That could be due to merge and acquisition processes in the past or any other reasons. You have most of workloads on-premises and extensive use of Windows OS. Active Directory is a standard identity service and you have multiple AD forests in different companies. Mixture of different networks with overlapping IP ranges prevents to establish trusts and establish collaboration across companies. Now, you are about to adopt different SaaS solutions or online services like Microsoft to improve collaboration in the cloud. It is not the secret: the only way to do achieve this was to configure synchronization using AD Connect. But after some research, you come across the
[following limitation](https://docs.microsoft.com/en-us/azure/active-directory/hybrid/plan-connect-topologies#multiple-forests-multiple-sync-servers-to-one-azure-ad-tenant).
<figure>
	<img src="{{ '/assets/img/2021-02-28/ADConnectUsnsupported.png' | prepend: site.baseurl }}" alt=""> 
	<figcaption>Multiple forests, multiple sync servers to one Azure AD tenant</figcaption>
</figure>

*Having multiple Azure AD Connect sync servers connected to the same Azure AD tenant is not supported, except for a staging server. It's unsupported even if these servers are configured to synchronize with a mutually exclusive set of objects. You might have considered this topology if you can't reach all domains in the forest from a single server, or if you want to distribute load across several servers.*

This is somehow expected if you recall what AD Connect historically is. ADC, previously known as DirSync and AAD Sync, is re-mastered product called Microsoft Identity and Access Management (FIM). FIM in turn is previously known as Identity Lifecycle Manager that was in turn successor of Microsoft Identity Integration Server 2003). That is, under the hood AD Connect still has 15+ years old pieces of code and inherited its architecture from that time. Metaverse as a set of tables in SQL Server database instance should contain information about all identities AD Connect work with. If you’re unable to provide access to source of identities (remote AD forest) you cannot bring them to Azure AD in a supported way. However, I saw environment built in this “not support but will work” way.  If you still want to be on a supported side and use single AD Connect instance, you would have to first complete AD migration project to move all identities to the single forest. If you ever participated in such huge never-ending projects, you would not want to do it again.

#### Azure AD Connect or OKTA 
If you had looked at Gartner quadrant in 2019 or before, then you probably would have chosen OKTA as much better and more reliable option.

<figure>
	<img src="{{ '/assets/img/2021-02-28/OKTA2019.png' | prepend: site.baseurl }}" alt="" width="500"> 
	<figcaption>Gartner Magic Quadrant for Access Management 2019</figcaption>
</figure>

(BTW, probably releasing AAD Connect cloud sync helped Microsoft to finally take the top place in Gartner Magic Quadrant 2020!)

<figure>
	<img src="{{ '/assets/img/2021-02-28/OKTA2020.png' | prepend: site.baseurl }}" alt="" width="500"> 
	<figcaption>Gartner Magic Quadrant for Access Management 2020</figcaption>
</figure>


AAD Connect cloud sync improved for the last year a lot comparing to its initial state. But back in late 2019 we didn’t know how fast it will evolve. We had to make a choice between purchasing OKTA or using new sync agent and initially we decided to go with OKTA. But shortly before performing PoC for OKTA, we were approached by our Microsoft Account Manager.  He shared information about new sync service, initially called AAD Connect cloud provisioning. Microsoft finally will have new light-weight agent very soon! Yes, it is in preview with limited (comparing to classic AD Connect) number of features, but it finally allows to synchronize objects from multiple AD domains that have no networking connectivity and trusts. 
As we already used Microsoft 365 services in several companies with some amount of E3 and E3 O365 licenses, that would allow us to save money for licenses. So that is a decision we had to make:
* Spend extra money for licenses and choose OKTA to get guaranteed result. In the future, however, requirement to cut IT costs would mean another project to migrate from OKTA
* Go with service that is currently in preview and experience unexpected problems being very early adopters. Something you probably never do for your production system. However, if have had succeeded, we would have got a working solution in scope of current budget planned for M365 services.

### Still missing block 
Well, We took the risky second option. Honestly, I regretted few times about this decision when the sync service was at its early stages. But after all that was a successful project and as a result, you are now reading this. This is what we’ve finally come to.

<figure>
	<img src="{{ '/assets/img/2021-02-28/AADDiagram.PNG' | prepend: site.baseurl }}" alt="" width="800"> 
	<figcaption>Synchronization and provisioning architecture</figcaption>
</figure>


This diagram was taken from [Protecting Microsoft 365 from on-premises attacks](https://techcommunity.microsoft.com/t5/azure-active-directory-identity/protecting-microsoft-365-from-on-premises-attacks/ba-p/1751754) blog post. I deliberately removed part related to provisioning from HR apps to Azure AD. This part has not yet been finished because of two reasons:

**Organizational:** user provisioning of any kind is much more complicated comparing to synchronization and therefore can slow down main task of providing quick access to global services through Azure AD

**Technical:** once hybrid users are in Azure AD, it is not possible to convert them to cloud-only in a supported way. This means hybrid users could be only mastered in Active Directory. There are conversion methods that might work like one described  [here](https://www.blogabout.cloud/2019/08/871/). There are also comments with similar solutions in [Allow Conversion of AD Synced Accounts to "In Cloud Only"](https://feedback.azure.com/forums/169401-azure-active-directory/suggestions/36479119-allow-conversion-of-ad-synced-accounts-to-in-clou) post on Azure feedback forum, however, the post itself marked only as “Planned”. So, it is not yet clear how to stop syncing user accounts from AD and switch them to be sourced from HR applications without affecting user access. Provisioning from HR services to multiple AD domain is actually possible according to [Plan cloud HR application provisioning](https://docs.microsoft.com/en-us/azure/active-directory/app-provisioning/plan-cloud-hr-provision#single-cloud-hr-app-tenant---target-multiple-child-domains-in-a-disjoint-active-directory-forest) article.

This would wrap up first introduction part of this blog series. Next will be more dedicated to solving issues from technical perspective. So, tech Azure AD funs, stay tune! 

##### In the parts of Bringing ‘em all in blog series we will cover the following topics:

* Understanding AAD Connect cloud sync schema and extending attributes
* Dealing with chaos in Active Directory
* Non-routable domains in UserPrincipalName attributes of AD users
* Users who are already being synced to another Azure AD tenant
* Shadow Azure AD tenants
* Administration delegation inside your central tenant
* Lack of conflict resolution for attributes and duplicate objects
