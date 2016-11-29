---
layout: post
title:  "Session Management Options for Sitecore 8 PaaS in Azure"
date:   2015-10-12 20:35:00
categories: Sitecore Azure
comments: true
---
Sitecore 8 introduces a significant shift in session management, as both private and shared session providers are introduced to fully support the CMS with xDB integration. If you're considering a PaaS model in Azure and have your own deployment strategy, keep reading. If you're using Sitecore's Azure module, you can pretty much stop here as the decision has been made for you. 

For load balanced on-premise solutions, session management is rather straight forward. The two primary options are either Sitecore's custom SQL or Mongo session providers. PaaS solutions in Azure slightly complicate the decision. Throw in Sitecore's xDB cloud edition and your options become more interesting.  In this post, we'll take a look at session management options specific to Sitecore 8 PaaS.

# Sitecore Azure Module

Sitecore's Azure module uses a custom In-Role Cache provider, defined within a global web.config transformation field labelled "do not edit!" within the client. This value will inject the provider within your Azure package's web.config prior to uploading and transitioning to a cloud service. 

# Custom Deployment Strategies

If you have your own in-house deployment strategy outside of the Azure module (such as Azure PowerShell scripts), you have a few options. You can proceed with the recommendation within the Azure module and use the in-role cache provider, or you can move forward with one of the standard providers more commonly used with on-premise solutions. 

## A Note on Using xDB Cloud Edition

While xDB Cloud Edition removes the necessity for managing multiple components of the reporting, processing, indexing layers and collection of xDB data, this can impact your choice for session management in Azure.

Sitecore's xDB Cloud Edition currently does not support a sessions database. If you choose xDB Cloud Edition, you're left looking elsewhere for a session state storage solution. 

## Session Provider Options

When considering Azure PaaS, we have three options:

	1. Mongo
	2. SQL
	3. In-Role Cache

### Mongo

Probably the simplest option, if you're using a hosting provider like MongoLabs. Creation of an additional Sessions database is simple and straight-forward. 

If using Sitecore's hosted xDB solution, this is less likely to be an acceptable solution given the cost. Paying fees for an externally hosted Mongo solution simply for a Sessions database is not worth the value in return versus our other two options.

### SQL

Sitecore's custom SQL session provider allows the ability for Sitecore to flush session data on the ```Session_End``` event. You can continue to use this method in Azure, but the preparation for deployment to Azure SQL involves an additional step than on-premise solutions. If you try to export the Sessions database to an SQL Azure .bacpac format, you'll run into an error during the export. In order to successfully export to a .bacpac, delete the ```CreateTables``` stored procedure using the statement below:

```DROP PROCEDURE [dbo].[CreateTables]```

Once the stored procedure has been removed. Proceed with transitioning to Azure SQL as you would with any SQL database. Thanks to Oleg Burov at Sitecore for providing direction using this method. 

#### Sitecore SQL Session Database != ASP.NET SQL Session Database

In my initial research in using Azure SQL for session management, I incorrectly assumed this method involved ASP.NET's standard SQL session management database. Typically, this is done by running a command similar to:

```Aspnet_regsql.exe's –sstype```

However, in doing so, you cannot export and use with Azure. Searching Microsoft's articles on the topic, you'll see that all support for using this method in Azure is <a href="https://azure.microsoft.com/en-us/blog/using-sql-azure-for-session-state/" target="_blank">no longer supported by Microsoft</a>:

<blockquote>
The code below is not supported by Microsoft, our support policy for session state using the SQL Azure database is stated as: ”Microsoft does not support SQL Session State Management using SQL Azure databases for ASP.net applications” in this Knowledge Base article.
</blockquote>
	
Stick with Sitecore's custom SQL session database and provider. The standard ASP.NET session database is both not supported by Microsoft and Sitecore's custom provider via ```Sesion_End```.

### In-Role Cache

The majority of Sitecore 8 PaaS implementations will rely upon the Sitecore 8 Azure module to deploy to the cloud. The Azure deployment content items contain a global web.config transformation which introduces Sitecore's in-role cache provider: ```Sitecore.Azure.SessionStateProviders.AzureInRoleCacheSessionStateProvider, Sitecore.Azure.SessionStateProviders```. You will only see this dependency upon installing the Azure module for Sitecore 8. 

### What About Redis Cache?

You might be tempted to look into Redis Cache. As tempting as this may seem, this is not an option for use with Sitecore 8. A Redis Cache provider has not been built to support the ```Session_End``` event required by xDB. Judging by the protection levels of the default <a href="https://msdn.microsoft.com/en-us/library/azure/dn690522.aspx" target="_blank">ASP.NET Session State Provider for Azure Redis Cache</a>, an extension of this to support Sitecore 8 would take quite a development effort (and rework of the default library).

# Closing Remarks

Session provider options will depend greatly on which deployment method you choose for your Sitecore 8 PaaS solution and whether or not you're using xDB Cloud Edition. 

Use the table below to assist in determining which method to use. 

<table rules="groups">
	<thead>
		<tr>
			<th style="text-align: left">Deployment Method</th>
			<th style="text-align: left">Mongo Solution</th>
			<th style="text-align: left">xDB Cloud Edition</th>
		</tr>
	<thead>
	<tbody>
		<tr>
			<td>Azure Module</td>
			<td>In-Role Cache</td>
			<td>In-Role Cache</td>
		</tr>
		<tr>
			<td>Deployment Scripts</td>
			<td>Mongo</td>
			<td>SQL Azure or In-Role Cache</td>
		</tr>
	</tbody>
</table>