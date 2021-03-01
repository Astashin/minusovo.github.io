---
layout: post
title:  "Bringing ‘em all in: implementing single Azure AD tenant  for enterprises"
date:   2021-02-28
description: On last Friday of January 2020, Microsoft announced general availability Azure AD Connect cloud sync feature. For us, as very early adopters of the new hybrid synchronization platform that was not something very special
---

<p class="intro"><span class="dropcap">O</span>n last Friday of January 2020, Microsoft announced general availability Azure AD Connect cloud sync feature. For us, as very early adopters of the new hybrid synchronization platform that was not something very special: we started to use it in production in early 2020 shortly after it was released in preview. That was challenging journey as one can expect to experiencing being brave adopting a service in its early stages. This blog series is about this story, technical problems and how I could work them around.</p>

Imagine, you're mid to enterprise level company in 2019 who heavily on Microsoft products on its on-premises systems. You're not just one monolithic firm, but holding that contains dozens of other smaller ones collected during merge and acquisition processes in the past. Maybe there was another story that led to the you architecture – multiple AD forests, mixture of networking with overlapping IP ranges or having other issues preventing to connect them all. Now, you're about to adopt different SaaS applications or online offering from Microsoft  and for this looking you way to implement hybrid configuration. You soon realize, the only way to do this being full Microsoft is to implement synchronization with AD Connect. After some research, you come across the following limitation.

<figure>
	<img src="{{ '/assets/img/2021-02-28/ADConnectUsnsupported.png' | prepend: site.baseurl }}" alt=""> 
	<figcaption>Multiple forests, multiple sync servers to one Azure AD tenant</figcaption>
</figure>

Having multiple Azure AD Connect sync servers connected to the same Azure AD tenant is not supported, except for a staging server. It's unsupported even if these servers are configured to synchronize with a mutually exclusive set of objects. You might have considered this topology if you can't reach all domains in the forest from a single server, or if you want to distribute load across several servers
This is somehow expected if you recall what AD Connect (ADC; previously known as DirSync and AAD Sync) historically is. ADC is re-mastered another old product of Microsoft called Microsoft Identity and Access Management (FIM; previously known as Identity Lifecycle Manager that was in turn successor of Microsoft Identity Integration Server 2003). That is, under the hood ADC still has 15+ years old pieces of code and inherited its design. Design that has your Active Directory as a primary source for user identities, components that you have additionally install and so on. Term like Metaverse, requirement to have SQL Server instance and others is a baggage you took from that old time. 
So what our options then, you might ask. You then might have a look at Gartner quadrant and realize, the only option for you would be to choose another vendor.

<figure>
	<img src="{{ '/assets/img/2021-02-28/OKTA2019.png' | prepend: site.baseurl }}" alt="" width="500"> 
	<figcaption>Magic Quadrant for Access Management 2019</figcaption>
</figure>

