---
layout: post
title:  "Upgrading to Sitecore 9"
date:   2018-11-01 20:00:00
categories: Sitecore
comments: true
---

Recently, we upgraded Sitecore form 8.1 update 3 to 9.0 update 1. Note this post deals with an XM instance. XP upgrades may follow a similar path, however, XP specific upgrade tasks are not covered here.

## Phases of a Sitecore Upgrade

- Phase 1: Planning and Identifying Scope
- Phase 2: Development and Validation
- Phase 3: Execution


# Phase 1: Planning and Identifying Scope

The general approach we took for the upgrade path was to execute in the following order:

1. Upgrade solution in source
2. Upgrade databases
3. Deploy updated solution


## Scope of Changes

- Updating all projects to .NET framework 4.7
- Updated all Sitecore references to 9.0 update 1 NuGet packages (latest)
- Replaced IoC container with Microsoft Dependency Injection Framework (1.0.0)
- New IoC container wire-up pipeline processors
- Modified config files for new roles-based format (no more switch master with web)
- New Application Initialization pipeline processor to replace WebActivator logic (IoC container wire-up dependency)
- Removal of testing index
- Update code for any Sitecore API breaking changes
- Solr configuration changes around placement of `documentOptions` XML node

## Prerequisites

Ensure your solution is clean to support an upgrade. In other words, no direct editing of Sitecore OOTB files.

- NuGet package references for all Sitecore dependencies
- Config patches for application-specific customizations
- No directly modified Sitecore files in source (especially config files)

# Phase 2: Development and Validation

## Step 1: Upgrade Solution

Goal: Upgrade your .NET solution against a vanilla Sitecore 9.0 update 1 set of databases

While Sitecore provides the ability to update the filesystem by using the wizard, the results of the upgrade will fail as new requirements prevent the site from loading. Specifically, changes around IoC configuration, wire-up of dependencies and roles-based transformations. As far as upgrading the solution goes, I recommend updating this manually against a vanilla Sitecore 9.0 update 1 set of databases. The goal of which is to ensure the Sitecore login page loads, you can see the vanilla instance content tree and the logs are free of errors. 

The following are application specific changes implemented while upgrading the .NET solution.

### 1. Invalid Internal Link Field Values

In our solution, we had internal link field values which had `<internalLink />` without any XML attributes. If left unchanged, publishing will fail during the upgrade. This issue is not limited to just our solution, as Sitecore has confirmed this a known issue. To resolve, they actually issued us a modified Sitecore.Kernel.dll, with changes limited to null checking during the building of internal links:

{% highlight c# %}
protected virtual string GetInternalUrl(Database database, string url, string itemID, string anchor, string queryString)
{
  Assert.ArgumentNotNull((object) database, nameof (database));
  Assert.ArgumentNotNull((object) url, nameof (url));
  Assert.ArgumentNotNull((object) itemID, nameof (itemID));
  Assert.ArgumentNotNull((object) anchor, nameof (anchor));
  Assert.ArgumentNotNull((object) queryString, nameof (queryString));

  // START: Added by Sitecore Support
  if (string.IsNullOrEmpty(url) && !ID.IsID(itemID))
	return string.Empty;
  // END

  Item obj = database.GetItem(url) ?? database.GetItem(new ID(itemID));
  if (obj == null)
	return string.Empty;
  if (obj.Paths.IsMediaItem)
	return this.GetMediaUrl(database, itemID);
  UrlString urlString = this.GetUrlString(obj);
  urlString.Parameters.Add(new UrlString(queryString).Parameters);
  urlString.Hash = anchor;
  urlString.Hash = new UrlString(queryString).Hash;
  return urlString.GetUrl();
}
{% endhighlight %}

An alternative solution is to execute a PowerShell script against your master database removing these invalid values. However, note this can be quite time consuming depending on the size of your tree.

{% highlight powershell %}
filter HasBrokenLink {
    param(
        [Parameter(Mandatory=$true,ValueFromPipeline=$true)]
        [Sitecore.Data.Items.Item]$Item,
        
        [Parameter()]
        [bool]$IncludeAllVersions
    )
    
    if($Item) {
        try {
            $brokenLinks = $item.Links.GetBrokenLinks($IncludeAllVersions)
        }
        catch {
            Write-Host $item.ID $item.Version.Number $item.Language.Name $item.Paths.FullPath
            
            $item.Fields.ReadAll()
            for ($index = 0; $index -lt $item.Fields.Count; $index++) {
                $field = $item.Fields[$index]
                if ($field) {
                    $fieldLink = [Sitecore.Data.Fields.FieldTypeManager]::GetField($field)
                    if ($fieldLink -is [Sitecore.Data.Fields.LinkField]) {
                        $isBad = $field.Value -eq "<link linktype=`"internal`" />"
                        if ($isBad) {
                            Write-Host "Removing invalid internal link field value for" $field.Name $field.Value
                            
                            $item.Editing.BeginEdit()
                            $field.Value = ""
                            $item.Editing.EndEdit()
                        }
                    }
                }
            }
        }
    }
}

$includeAllVersions = $true
$database = "master"
$root = Get-Item -Path (@{$true="$($database):\content\home"; $false="$($database):\content"}[(Test-Path -Path "$($database):\content\home")])
Get-ChildItem -Path $root.ProviderPath -Version * -Language * -Recurse | HasBrokenLink -IncludeAllVersions $includeAllVersions

Write-Host "[done]" -foreground green
{% endhighlight %}

### 2. Deep Linking Always Enabled for AddItemLinkReferences Pipeline Processor

Disable deep linking, as this will be enabled by default with 9.0 update 1. This is significant for those who configure publishing to trigger based on submit actions in workflow, especially if you're publishing with related items. To disable this, patch in the following to disable deep scanning.

{% highlight xml %}
<processor type="Sitecore.Publishing.Pipelines.GetItemReferences.AddItemLinkReferences, Sitecore.Kernel">
  <DeepScan>false</DeepScan>
</processor>
{% endhighlight %}

### 3. IoC Container Registration via Pipeline Processor

No longer should this be executed via application start, as application-specific dependency registration will be appended to the container Sitecore is initializing at startup. This means registration via pipeline processors. Wire-up the registration processor using a patch file like the one below. There are plenty of articles covering dependency injection with Sitecore 9, so I won't go into specifics here.

{% highlight xml %}
<?xml version="1.0"?>
<configuration>
	<sitecore>
		<pipelines>
			<initialize>
				<processor type="My.Registration.ApplicationInitialization, My.Website" resolve="true" />
			</initialize>
		</pipelines>
	</sitecore>
</configuration>
{% endhighlight %}

### 4. Resolving Wildcard Items in Sitecore MVC

Read <a href="https://jammykam.wordpress.com/2017/08/02/sitecore-mvc-context-item/" target="_blank">this thoughtful post on resolving of the context item</a> in Sitecore MVC. Specifically for our solution, wildcard items were not using the proper mvc.getPageItem processor.

### 5. Solr Search Logs Grow Excessively

Also a known issue with Sitecore 9 update 1, we needed to suppress warnings in our search logs. Use the following patch

{% highlight xml %}
<?xml version="1.0"?>
<configuration xmlns:patch="http://www.sitecore.net/xmlconfig/">
  <sitecore>
	<log4net>
		<appender name="SearchLogFileAppender">
			<filter type="log4net.Filter.StringMatchFilter">
				<regexToMatch value="Processing Field Name|Resolving Multiple Field found on Solr Field Map. No matching solr search field configuration on index field name" />
				<acceptOnMatch value="false" />
			</filter>
		</appender>
	</log4net>
  </sitecore>
</configuration>
{% endhighlight %}

### 6. Disable Device Detection

We don't use this feature. It ships enabled, so we chose to disable using the follow patch:

{% highlight xml %}
<configuration xmlns:patch="http://www.sitecore.net/xmlconfig/">
	<sitecore>
		<settings>
			<setting name="DeviceDetection.Enabled">
				<patch:attribute name="value">false</patch:attribute>
			</setting>
		</settings>
	</sitecore>
</configuration>
{% endhighlight %}

### 7. Reduce UpdateListOperationsAgent Job Frequency

We don't use this feature, yet the job runs multiple times a minute. Added this to reduce the entries in the logs

{% highlight xml %}
<scheduling>
	<agent type="Sitecore.ListManagement.Operations.UpdateListOperationsAgent, Sitecore.ListManagement">
		<patch:attribute name="interval">01:00:00</patch:attribute>
	</agent>
</scheduling>
{% endhighlight %}

### 8. Increase Solr Timeout

With Sitecore's default Solr timeout, index rebuilds will fail during the optimization step.

Use the following patch to update to 10 minutes

{% highlight xml %}
<configuration xmlns:role="http://www.sitecore.net/xmlconfig/role/" xmlns:patch="http://www.sitecore.net/xmlconfig/" xmlns:set="http://www.sitecore.net/xmlconfig/set/">
	<sitecore>
		<settings>
			<!-- Solr timeout set to 10 minutes (allows for index optimization agent to complete without error) -->
			<setting name="ContentSearch.Solr.ConnectionTimeout">
				<patch:attribute name="value">600000</patch:attribute>
			</setting>
		</settings>
	</sitecore>
</configuration>
{% endhighlight %}

### 9. Reduce Frequency of Indexing State Switcher Agent

This job runs more often than it needs to for us, filling the logs with unnecessary details. Add this transform:

{% highlight xml %}
<configuration xmlns:role="http://www.sitecore.net/xmlconfig/role/" xmlns:patch="http://www.sitecore.net/xmlconfig/">
	</sitecore>
		</scheduling>
			<agent type="Sitecore.ContentSearch.SolrProvider.Agents.IndexingStateSwitcher, Sitecore.ContentSearch.SolrProvider">
				<patch:attribute name="interval">00:10:00</patch:attribute>
			</agent>
		</scheduling>
	</sitecore>
</configuration>
{% endhighlight %}

## Step 2: Upgrade databases

Goal: Upgrade your content against a vanilla Sitecore 9.0 update 1 website

Point a vanilla Sitecore 9.0 update 1 website at your pre-upgrade databases. Follow the installation and upgrade steps within the upgrade documentation. This involves running the upgrade wizard. Do NOT check off upgrade filesystem. Upgrading the filesystem may cause failures at the very end of the upgrade. If this happens, you won't know for sure that the upgrade has completed successfully. Upon completion, verify that you can see your content tree through the Sitecore client. At the end of this step, you will have your databases upgraded to Sitecore 9.0 update 1.

## Step 3: Deploy updated solution

Goal: Validate your upgraded .NET solution against upgraded databases

It is here where you will confirm everything has been upgraded properly. This step can be the most time consuming, as an iterative development approach to bug fixing and testing will be required to ensure the solution has been fully upgraded. Be sure to scan the logs for errors and all editing capabilities are working as expected.

# Phase 3: Execution

Having successfully planned and executed the upgrade in a development environment, validating your approach, you must then execute in a production environment. Draft and follow an upgrade guide. 

The upgrade itself will requie a period of time where a content freeze is needed to prevent editors from updating content that could be lost during the upgrade. Be sure to backup your databases before upgrading. Having your existing Siteocre website running alongside the newly upgraded website provides a clean and simple rollback strategy in the event of failures during execution. Rebuilding of indexes and the links database should be executed as well before declaring success.