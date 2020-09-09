---
layout: post
title:  'Integrating Chef with ServiceNow - Part 1 - Introduction'
tags: [ Chef Automate, ServiceNow ]
excerpt: 'An introduction to the various ways you can integrate Chef with ServiceNow. Part 1 of a technical blog series'
---

When managing infrastructure and other resources with Chef you are given an incredible insight into your managed nodes. From installed packages to configuration values to system data like IPs and FQDNs, the data gathered by Chef can be leveraged in essentially any "single pane of glass" system your company uses. It becomes as simple as answering the question "what data do I want?".


### What data do you want? 

If you're like me, you've answered that you want all of the data. And why wouldn't you? For ChefConf Online, I contributed a session called "Use Data, You Must: Leveraging Chef Data in ServiceNow". It is in this session that I discuss the various ways you can integrate Chef with ServiceNow to get the data you want. What I do not cover in this session though is the technical details. For a one hour session, there simply isn't enough time to cover all of the gritty details. But what I do cover is the types of data you can glean from Chef and then feed into ServiceNow and the different ways in which you can do that. 

If you haven't, I would recommend starting with watching this session for some background. 

[![Use Data, You Must: Leveraging Chef Data in ServiceNow](/assets/images/posts/2020/chef-with-servicenow-part1/use-data-you-must.png)](https://eviacms.evia.events/embed/insights?mid=d96fb77a-fc40-4a27-a1ee-a1ccfbfe0303&startTime=0&endTime=2881)

### What's going to go down in this blog series?

In this blog series, I will be covering the following potential Chef + ServiceNow integration setups: 
- Incidents
- Changes
- CMDB


These technical posts will cover the steps to take to get a working integration between the two systems. There are many ways you can go about this but my hope is to provide simple tutorials to get you going for each integration point.

### Stay tuned for Part 2 where I will cover the incident integration between Chef-Client failures and ServiceNow

