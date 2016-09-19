---
layout: page
title: Debussyl&#58; A Database-Defined Network
description: "Debussy&#58; A Database-Defined Network"
---
{% include JB/setup %}


Debussy is a software-defined networking (SDN) controller that uses a standard SQL database to represent the network.  _Why a database?_ SDN fundamentally revolves around data representation--representation of the network topology and forwarding, as well as the higher-level abstractions useful to applications.

In Debussy, the entire network control infrastructure is implemented within a SQL database.  Abstractions of the network take the form of _SQL views_ expressed by SQL queries that can be instantiated and extended on the fly.  To allow multiple simultaneous abstractions to collectively drive control, Debussy automatically _orchestrates_ the abstractions to merge multiple views into a coherent forwarding behavior.

<br/>

## Get Started ##

Download [Debussy]({{site.url}}download) or clone our [GitHub repository](http://github.com/debussy-net/debussy).  Then try the [Walkthrough]({{site.url}}walkthrough).

Or develop your own applications.  Read our [Developer Guide]({{site.url}}manual) and browse through the [API](api/annotated.html).
