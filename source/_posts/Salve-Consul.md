---
title: Salve, Consul
date: 2017-11-11 13:36:47
tags: consul
---
When choosing the appropriate tool to get the job done, it is usually hard to foresee how big the impact of the decision is going to be. Some tools will cause us to curse ourselves for not having anticipated all the drawbacks, inconsistencies and bugs they carry along into your stack. Some just do their job and nothing more.
But then there are those special kind of growers that just keep facinating us with the amount of use cases they help us solving easily. 

HashiCorp Consul is one of these special kind of magic tools that not only helped me setting up service discovery solutions - which it is most known for. When I first decided to use Consul in production, I didn't even realize it has a second major feature: an easy to use, robust and well-documented key-value store.
It can serve many purposes - ranging from laying the groundwork for a sophisticated configuration-management to implementing flat-out hacks in your average garbage microservice.


In this series I want to share with you some of the use cases that I solved with using Consul's key-value store. All of them could have been solved with other technologies like Redis or just a simple git repository on a shared file system, but as Consul usually proves to be a single point of failure when used as a discovery server, we might just use it for other mission critical use cases.


Setting it up
-------------

Version management
------------------

Configuration management
------------------

Optimistic locking hacks
------------------

