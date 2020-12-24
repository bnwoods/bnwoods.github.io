---
layout: post
title:  'Integrating Chef with ServiceNow - Part 2 - Incidents'
tags: [ Chef Automate, ServiceNow ]
excerpt: 'An introduction to integrating Chef-Client failures with ServiceNow incidents - Part 2 of a technical blog series.'
featured: true
hidden: true
---

Very early in everyone’s Chef journey, we come to realize that keeping on top of failures is both an important problem to solve but could potentially be a hard problem to solve. Chef Automate gives a great at-a-glance solution for understanding the health of your estate, but what do you do when you need actionable insights? This is where the integration points between Chef and ServiceNow can really help.

As previously mentioned in the introduction to this blog series, Chef has put together some great integrations between Chef Automate and ServiceNow. But they’ve also done one better: they have given users the power to extract failure data from Automate using a webhook so it can be used in any way you desire. In our case, it was important to get this failure data into the hands of the people that could do something about it. We do not have a singular team responsible for Chef things; instead, we have given the power to those that are most equipped for it. Why not use the collective expertise of all your in house teams when dealing with configuration? Network Engineers should be able to set network configuration on systems – it is, after all, their area. Database Administrators should be able to set up their DB configs. The same with Systems Engineers and monitoring teams. Why can’t they be in charge of the things they’re already in charge of? Why add a bottleneck unnecessarily? 

But, it’s all of these reasons why getting incidents out became more complex. How do we distinguish between failures? The answer became quite simple: send information from Chef Automate to ServiceNow and have ServiceNow do the heavy lifting. Because of our Chef responsibility structure, using the out of the box solution moved out of our grasp. We needed to build something.

We have all the data.

Now let’s make it actionable.

### Caveats

Your mileage may vary. The below steps include the pieces we used to put this together, but this will vary wildly based on organization, ServiceNow setups, etc. I have also not included exact steps for sub-flow creation etc. This is simply meant to help new users or users trying to develop a similar solution to know what process to follow.

### Shameless plug on my coworkers

I think here is a good place to plug my incredible co-workers who really allowed this to happen. By all of us pooling our collective knowledge we were able to create a nice integration that works well for us. When this was written there wasn’t a built-in integration, instead, we built this using Chef Automate’s notification engine using the generic webhook.

## Step 1: Define your CIs

For this to be successful, we had to take a hard look at who cares about Chef Infra Client failures, why they care about them, and how we can programmatically tell ServiceNow who that team is. For us, this information lives in cookbook metadata. To get that, we need to regularly get the information from the Chef Server into ServiceNow, making our Cookbooks CIs. Chef now has a supported integration between Chef Automate and ServiceNow CMDB which allows you to do exactly this. In our case, pre-integration, we had to get creative with workflows and scheduled jobs – but using this out of the box integration will get what you need.
The other important part of this is defining the support group for these cookbook CIs. Without a support group, all incidents need to go to an Operations Center or Tier 1 group. Based on your org, that will be your determination to make on how incidents get routed to teams. For us, we define support groups on cookbooks.


## Step 2: Create an endpoint in ServiceNow

In your “Scripted REST API” area in ServiceNow, create an endpoint. We have an API definition of Chef with “sub” areas below, the client failure one being called “clientFailures” meaning your resource path should look something like /api/namespace/chef/clientFailures. You will need an HTTP method of POST and for your script, you will want something like the following:

<pre><code class="language-javascript">
(function process(/*RESTAPIRequest*/ request, /*RESTAPIResponse*/ response) {

	var payload = JSON.parse(request.body.dataString);
	if (!payload){
		response.setStatus(400);
		return 'Unable to process request. Request payload was not well-formed JSON or an error was encountered in parsing the JSON payload. Please check your input and try again.';
		
	} else {
		
		if (payload.attachments[0].fields[1].value.startsWith("insert_cookbook_prefix_here") ){
			// Use a regex to grab the run id so it can be used for the event message key	
			var rex = new RegExp('(?:\/runs\/)([A-Za-z0-9]*\-[A-Za-z0-9]*\-[A-Za-z0-9]*\-[A-Za-z0-9]*\-[A-Za-z0-9]*)(?:\|)', 'g');
			var str = payload.text;
			var message = str.split(rex);
			
			var chefUrl1 = str.slice(1).split('|');
			var chefAutoURL = chefUrl1[0];
			
			// Get the shortened error message
			var errorArray = payload.attachments[0].text.split('\n');
			var errorString = errorArray.join('\r\n\t');
			//errorString = errorString.replace('"', "'");
			errorString = errorString.replace('```Error:', 'Error:');
			errorString = errorString.replace('```', ' ');
			
			// Set the cookbook name and additional info variables for later use
			var cookbook, node = '';
			for (var i = 0; i < payload.attachments[0].fields.length; i++){
				// Get the cookbook name
				if (payload.attachments[0].fields[i].title == 'cookbook::recipe' || payload.attachments[0].fields[i].title == 'Cookbook::Recipe'){
					var ckArray = payload.attachments[0].fields[i].value.split("::");
					cookbook = ckArray[0];
				}
				// Get the node name
				if (payload.attachments[0].fields[i].title == 'node' || payload.attachments[0].fields[i].title == 'Node'){
					node = payload.attachments[0].fields[i].value;
				}
			}
			
			// Make sure the cookbook CI has an assigned support group
			var cbci = new GlideRecord('u_chef_cookbook');
			if (cbci.get('name',cookbook) && cbci.support_group.hasValue()){

				var addInfo = {
					"name": cookbook,
					"error": errorArray[0],
					"URL": chefAutoURL,
					"chefnode": payload.attachments[0].fields[0].value
				};
				

				// Create an event with basic required info
				var Event = new GlideRecord('em_event');
				Event.source = 'Chef Automate';
				Event.node = node;
				Event.message_key = message[1];
				Event.type = 'Chef-Client Failure';
				Event.resource = 'Chef';
				Event.error_msg = errorArray[0];
				Event.additional_info = JSON.stringify(addInfo);
				Event.severity = 4;
				Event.resolution_state = 'New';
				Event.ci_identifier = cookbook;

				if ( errorString.length < 4096 ) {
					Event.description = errorString;
				} else {
					Event.description = "See Attached Log for more details.";
				}
				
				// Let's go ahead and create the event now
				if (Event.insert()){

					// Attach error log.
					var gsa = new GlideSysAttachment();
					var attachmentId = gsa.write(Event, "ErrorLog.txt", 'text/plain', errorString);
					
					response.setStatus(200);
					return "Request processed, event record inserted. See ServiceNow event management for more details.";
				} else {
					response.setStatus(400);
					return "Request processed, but an error occurred with event creation. See ServiceNow system logs for more information.";
				}
			} else {
				response.setStatus(200);
				return "Request processed, but no event record was inserted due to cookbook not having a support group assigned.";
			}
		} else {
			response.setStatus(200);
			return "Request processed, but no event record was inserted due to cookbook name.";
		}
	}

})(request, response);
</code></pre>

Basically, the above javascript takes the payload from Chef Automate sent to the REST endpoint we just created, determines if it is one of our internally developed cookbooks (indicated by the cookbook prefix), determines the error/failure from the payload, the cookbook name, and creates an event if there isn’t already one. Additionally, the failure output is attached to the event.


## Step 3: Creating the alert correlation rule

Now that you have events, you need to correlate them. After all, if a single cookbook fails across 4,000 nodes you definitely do not want 4,000 incidents. Create an alert correlation rule with any descriptive name you see fit and give it the following filter conditions for the primary alert:
- Source is Chef Automate
- Type is Chef-Client Failure
- Configuration item.Support group is not empty

Secondary alert should have the following information:
- Source is Chef Automate
- Type is Chef-Client Failure
- Configuration item.Support group is not empty

Finally, for Relationship Type, you’ll select “Same CI or Node” and Time Difference in Minutes for us is 1,440.


## Step 4: Setup a sub-flow to create your incident

You will want to have a sub-flow create incidents from these events. This is where we use the support group for the cookbook CI to determine the assignment group.

## Step 5: Turn on the chef-client failure notifications in Chef Automate

The least involved part of this process is turning on the notification in Chef Automate. Depending on the level of output you want here, you can use either the Webhook option or the Slack option (the Slack option just includes some formatting stuff in the payload and cuts out the full stack trace).

![Setting Up the Notification in Chef Automate](/assets/images/posts/2020/chef-with-servicenow-part2-incidents/automate-settings.png)