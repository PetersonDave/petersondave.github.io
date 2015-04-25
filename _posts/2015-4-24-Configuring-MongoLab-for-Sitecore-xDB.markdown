---
layout: post
title:  "Configuring MongoLab for Sitecore xDB"
date:   2015-04-24 00:20:00
categories: Sitecore Mongo xDB
comments: true
---

The goal of this post is to outline the steps to connect a Sitecore instance to MongoLab for xDB tracking and analytics. With the introduction of the xDB, tracking and analytics have moved from a dedicated SQL instance with DMS, to a fire-and-forget model in xDB with Mongo.

There is a very helpful article on Sitecore's main site outlining the steps to <a href="http://www.sitecore.net/learn/blogs/technical-blogs/edvin-eshagh/posts/2015/03/deploying-sitecore-8-on-azure-website.aspx" target="_blank">deploy Sitecore 8 on Azure</a>, with references to MongoLab. Reference that article for a full walk-through to deploy to Azure.

<a href="https://mongolab.com/home" target="_blank">MongoLab</a> offers a terrific solution if you're looking to move your data collection to a hosted offering, one in which you can easily scale without having to worry about in-house support and maintenance. This holds especially true for those considering moving to a cloud-hosted solution for Sitecore content delivery, such as <a href="http://azure.microsoft.com/en-us/" target="_blank">Microsoft Azure</a>.

## Create a MongoLab Account

Go to <a href="http://www.mongolab.com" target="_blank">www.mongolab.com</a> and create an account. You may choose a pricing plan, or for those of you who are testing this configuration, move forward with a Sandbox environment with data allocations of up to 500 MB. 

## Create MongoLab Databases

Creating each database is simple. Click on the <em>Create New</em> button and follow the options to setup and configure your database. 

I chose Windows Azure for the cloud provider, since I'm hosting my sandbox Sitecore 8 instance in Azure. I want to also keep my MongoLab deployment instances in the same location as my cloud service Sitecore 8 instance (East US) and chose the <em>Single-Node</em> plan, as this is simply a Sandbox implementation. 

You may choose from one of the many options on the <em>Replica set cluster</em> side, as these options offer complete failover support for production instances. Note that after choosing the initial configuration, scaling should not be a problem if you eventually need to expand your MongoLab instance. 

Choose version 2.6.x or later if using Sitecore 8. 

For each database name, follow a similar naming convention for all of your databases. Remember these database names, as these will be applied to both your CE and CD connection strings.

![create new subscription](/assets/images/mongo-2.png)

For this example, I created entries for each of the following databases:

* Session
* Analytics <em>(required for content delivery)</em>
* Tracking Contact  <em>(required for content delivery)</em>
* Tracking Live  <em>(required for content delivery)</em>
* Tracking History  <em>(required for content editing)</em>

<em>Note:</em> See Sitecore's recommendation for configuring <a href="https://doc.sitecore.net/products/sitecore%20experience%20platform/xdb%20configuration/walkthrough%20configuring%20a%20shared%20session%20state%20database%20using%20the%20mongodb%20provider" target="_blank">MongoDB as your session provider</a>

After creating your databases, you should see a similar list of MongoDB deployments:

![mongo deployments](/assets/images/mongo-1.png)

## Security

Much like configuring SQL Server databases, you will need to create a user for each database. You can create new users by clicking on the <em>Add database user</em> button. Choose a password and set read/write access. Remember your password, as this will be applied to the Mongo connection URI.

![mongo security](/assets/images/mongo-5.png)

## Setup Connection Strings

Upon successfully creating your databases and related users, you can click on each database instance to see examples on how to connect to your new MongoLab database:

![mongo connection strings](/assets/images/mongo-3.png)

Copy the URI from the <em>To connect using a driver via the standard URI</em> and update you connection strings file. Your connection strings should resemble the following:

{% highlight xml %}
<?xml version="1.0" encoding="utf-8"?>
<connectionStrings>
  <!-- SQL connection strings -->
  <add name="core" connectionString="Data Source=DPETERSON;Initial Catalog=sc80rev150121AzureSitecore_core;Integrated Security=False;User ID=sitecore;Password=password" />
  <add name="master" connectionString="Data Source=DPETERSON;Initial Catalog=sc80rev150121AzureSitecore_master;Integrated Security=False;User ID=sitecore;Password=password" />
  <add name="web" connectionString="Data Source=DPETERSON;Initial Catalog=sc80rev150121AzureSitecore_web;Integrated Security=False;User ID=sitecore;Password=password" />
  <add name="reporting" connectionString="Data Source=DPETERSON;Initial Catalog=sc80rev150121AzureSitecore_reporting;Integrated Security=False;User ID=sitecore;Password=password" />

  <!-- MongoLabs connection strings -->
  <add name="session" connectionString="mongodb://dpeterson:password@ds060977.mongolab.com:60977/sc80rev150121azure_session" />
  <add name="analytics" connectionString="mongodb://dpeterson:password@ds060977.mongolab.com:60977/sc80rev150121azure_analytics" />
  <add name="tracking.live" connectionString="mongodb://dpeterson:password@ds060977.mongolab.com:60977/sc80rev150121azure_tracking_live" />
  <add name="tracking.history" connectionString="mongodb://dpeterson:password@ds062097.mongolab.com:62097/sc80rev150121azure_tracking_history" />
  <add name="tracking.contact" connectionString="mongodb://dpeterson:password@ds060977.mongolab.com:60977/sc80rev150121azure_tracking_contact" />
</connectionStrings>
{% endhighlight %}

## Test Your Connection

From there, your instance (whether cloud-service based or on-prem) will spin up and communicate with your new MongoLab databases. At that point, your MongoLab database collections will start to auto-generate, just as they would with any non-hosted mongo solution. Looking back at the Analytics MongoLab database, for example, you should see collections resembling the following:

![mongo collections](/assets/images/mongo-4.png)

Total setup time can be anywhere from 15-30 minutes. MongoLab's interface is quite simple and makes configuration a breeze. 